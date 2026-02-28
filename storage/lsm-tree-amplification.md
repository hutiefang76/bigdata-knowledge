# LSM-Tree 读放大、写放大、空间放大 — 原理与数据湖演进

> HBase → Iceberg → Paimon+Flink → Doris 各系统如何站在前人肩膀上解决放大问题

---

## 零、LSM-Tree 是什么，为什么它必须"放大"

### 0.1 先理解问题：磁盘最怕什么

```
机械硬盘 (HDD):
  顺序写: ~200 MB/s
  随机写: ~1 MB/s       ← 差 200 倍

固态硬盘 (SSD):
  顺序写: ~3000 MB/s
  随机写: ~500 MB/s     ← 差 6 倍，且随机写加速 SSD 磨损
```

传统数据库（MySQL InnoDB、PostgreSQL）用 B+Tree：

```
B+Tree 写入一条记录:
  1. 读取目标 Page (通常 16KB) ← 随机读
  2. 修改 Page 中的几十字节
  3. 写回整个 Page            ← 随机写
  4. 可能触发页分裂 → 再写几个 Page

每次写入 = 多次随机 IO
当写入量达到每秒数万~数十万条时，B+Tree 的随机 IO 成为瓶颈
```

### 0.2 LSM-Tree 的核心设计：拿"放大"换"速度"

LSM-Tree (Log-Structured Merge-Tree) 由 Patrick O'Neil 于 1996 年提出，核心思想：

> **把所有随机写变成顺序写，代价是后台必须做合并（Compaction），合并带来放大。**

```
B+Tree:  写入时就把数据放到"正确位置" → 随机写 → 慢，但读快
LSM-Tree: 写入时只追加到内存/文件末尾 → 顺序写 → 快，但数据无序，读要多查几个地方
```

这不是设计缺陷，而是**有意为之的 trade-off**。

### 0.3 LSM-Tree 完整写入流程

```
Step 1: 写入 MemTable（内存，红黑树/跳表，有序）
        ┌──────────────────────┐
        │  MemTable (sorted)   │  ← 所有写入先到这里，纯内存操作
        │  key=A → v3          │
        │  key=B → v1          │
        │  key=C → v2          │
        └──────────┬───────────┘
                   │ 满了 (如 64MB)
                   ↓
Step 2: Flush 为 SSTable（Sorted String Table，不可变有序文件）
        ┌──────────────────────┐
        │  L0/sst_001.dat      │  ← 一次顺序写，整个文件有序
        │  [A:v3, B:v1, C:v2]  │
        └──────────────────────┘
        ┌──────────────────────┐
        │  L0/sst_002.dat      │  ← 又一次 Flush
        │  [A:v5, D:v1, E:v1]  │     注意：A 出现了两次（v3 和 v5）
        └──────────────────────┘
                   │ L0 文件太多了
                   ↓
Step 3: Compaction（后台合并）
        读取 sst_001 + sst_002
        → 合并排序 → A 取最新 v5，B/C/D/E 保留
        → 写出新文件 sst_003 到 L1
        → 删除 sst_001, sst_002

        这一步就是"放大"的来源：
        sst_001 的数据被读出→合并→重新写入 sst_003
        同样的数据被写了两次（甚至更多次，因为 L1→L2 还会再合并）
```

### 0.4 为什么必须 Compaction — 不合并会怎样

```
假设从不做 Compaction:
  写入 100 万次 → 产生 100 万个 SST 文件 (每次 Flush 一个)

查询 key=X:
  → 不知道 X 在哪个文件
  → 必须从最新到最旧逐个查: sst_1000000, sst_999999, ..., sst_1
  → 最坏情况: 扫描全部 100 万个文件
  → 读放大 = 100 万倍
  → 完全不可用

空间问题:
  → key=A 被更新了 1000 次，100 万个文件中有 1000 个包含 A 的旧版本
  → 999 个是垃圾，但没人清理
  → 空间放大 = 1000x
```

**所以 Compaction 不是可选的，是必须的。放大是"维持可用性"的代价。**

### 0.5 三种放大的根因图

```
                        LSM-Tree 的设计决策
                               │
              ┌────────────────┼────────────────┐
              │                │                │
       写入只追加不覆盖    数据分散在多层文件    旧版本不立即删除
              │                │                │
              ↓                ↓                ↓
      Compaction 必须合并  查询必须多文件查找   过期数据占空间
      同一数据反复读写      才能拿到最新值        直到 Compaction 清理
              │                │                │
              ↓                ↓                ↓
          写放大            读放大            空间放大
    (Write Amplification) (Read Amplification) (Space Amplification)
```

### 0.6 LSM-Tree vs B+Tree 定量对比

| 指标 | B+Tree (InnoDB) | LSM-Tree (RocksDB) |
|------|----------------|-------------------|
| 随机写吞吐 | ~5,000 TPS | ~50,000+ TPS |
| 顺序写吞吐 | ~20,000 TPS | ~200,000+ TPS |
| 点查延迟 | O(log N) 稳定 | O(L) L=层数，有波动 |
| 范围查询 | 快（B+Tree 叶节点有序链表） | 需合并多层（较慢） |
| 写放大 | 1~2x（页写入） | 10~30x（多层 Compaction） |
| 空间利用率 | ~50-70%（页分裂碎片） | ~80-95%（Compaction 后紧凑） |

**结论**：LSM-Tree 用 10~30 倍写放大，换来 10 倍以上的写入吞吐。在写密集型场景（日志、IoT、CDC、时序数据），这个 trade-off 是值得的。

### 0.7 两种 Compaction 策略

#### Leveled Compaction（以读优先）

```
L0: [sst_1] [sst_2] [sst_3]     ← key 范围可重叠
L1: [sst_a] [sst_b] [sst_c]     ← 同层 key 范围不重叠！
L2: [sst_x] [sst_y] [sst_z]     ← 同层 key 范围不重叠！

Compaction 规则:
  L0 的 sst_1 与 L1 所有 key 范围重叠的文件合并
  → 可能要重写 L1 的多个文件
  → 写放大高 (一个 L0 文件可能触发重写整层 L1)

读取优势:
  L1/L2 每层只需查1个文件 (key 不重叠，二分定位)
  → 读放大低
```

#### Tiered Compaction（以写优先）

```
L0: [run_1] [run_2] [run_3]     ← 每层允许多个 sorted run
L1: [run_4] [run_5]             ← key 范围可重叠！
L2: [run_6]

Compaction 规则:
  同层积累 T 个 run 后，合并为1个 run 推到下层
  → 每个 run 只被写一次
  → 写放大低

读取劣势:
  每层多个 run，key 范围重叠
  → 每层都可能要查多个 run
  → 读放大高
```

#### 不可能三角

```
        写放大 低
           /\
          /  \
         /    \
        / 不可能 \
       /  三角    \
      /____________\
  读放大 低      空间放大 低

Leveled:  读低 + 空间低 → 写高
Tiered:   写低           → 读高 + 空间高
Universal: 动态平衡，在三角内找最优点
```

**这就是为什么后续所有系统（HBase/Iceberg/Paimon/Doris）都在这个三角内做优化，而不是消除放大——放大是 LSM-Tree 换取写入性能的本质代价。**

---

## 一、用具体例子讲清三种放大

### 1.1 写放大 — 改1行，磁盘写了100行

**场景**：用户表 1 亿行，按 user_id 分区，每个 Parquet 文件 128MB。

```sql
UPDATE user_table SET phone='13900001111' WHERE user_id = 12345;
```

**Iceberg COW 模式下**：

```
原始文件: user_part_001.parquet (128MB, 100万行)
                 ↓
找到 user_id=12345 所在文件
                 ↓
读取整个 128MB → 改1行 → 写出新 128MB
                 ↓
实际写入: 128MB / 有效变更: ~100B
写放大 ≈ 100万倍
```

**Paimon LSM 模式下**：

```
写入 (user_id=12345, phone='13900001111') → MemTable (内存)
                 ↓ (攒够 buffer 或 Checkpoint 触发)
Flush → L0 新 SST 文件 (~KB级)
                 ↓ (后台 Compaction)
L0 + L1 合并 → 新 L1 文件
                 ↓
实际写入: 数KB (Flush) + Compaction 再写一次
写放大 ≈ 2~10倍
```

**HBase 高频更新同一行**：

```java
// 10000次更新同一个 rowkey
for (int i = 0; i < 10000; i++) {
    Put put = new Put(Bytes.toBytes("row_001"));
    put.addColumn(CF, COL, Bytes.toBytes("value_" + i));
    table.put(put);
}
// 10000次写入 → MemStore → 多次 Flush → 多个 HFile
// Major Compaction 把所有 HFile 合并，只保留最新1条
// 磁盘实际写: 10000次 Flush + N次 Compaction 重写
// 有效数据: 1行
```

### 1.2 读放大 — 查1行，磁盘读了20个文件

**场景**：Paimon 主键表，持续流写入，Compaction 没跟上。

```sql
SELECT * FROM orders WHERE order_id = 'ORD_20260228_001';
```

**未及时 Compaction**：

```
L0: sst_100 ~ sst_80  (20个文件, key 范围全部重叠!)
L1: sst_50 ~ sst_79   (30个文件)
L2: sst_01 ~ sst_49   (49个文件)

查 order_id = 'ORD_20260228_001':
  → L0 的20个文件 key 范围重叠，每个都可能包含该 key
  → 逐个打开，读 Bloom Filter 判断
  → 然后 L1 找1个，L2 找1个
  → 为了1行数据，打开了 20+ 个文件

读放大 ≈ 20x ~ 50x
```

**Compaction 充分时**：

```
L0: (空)
L1: sst_01 ~ sst_10 (key 不重叠，有序)

查 order_id:
  → 二分定位到 sst_05 → Bloom Filter → Data Block → 返回
  → 只打开1个文件

读放大 ≈ 1x
```

### 1.3 空间放大 — 存1份数据，磁盘占了3份

**场景**：频繁 DELETE 但不做 Compaction。

```sql
DELETE FROM orders WHERE create_time < '2026-01-28';
```

**LSM-Tree 中 DELETE 不是真删除**：

```
原始数据:  L2 中 1000万条订单 (500MB)
执行DELETE: 写入 tombstone 到 L0 (几KB)

磁盘状态:
  L0: tombstone "删除 create_time < 2026-01-28"
  L2: 原始1000万条仍完整存在

有效数据: 700万条 (~350MB)
磁盘占用: 500MB + tombstone
空间放大 = 500MB / 350MB ≈ 1.43x
```

**Tiered Compaction 多版本极端情况**：

```
同一 key 更新5次，每次在不同 sorted run:

Run 1: (key=A, v1)  ← 最旧
Run 2: (key=A, v2)
Run 3: (key=A, v3)
Run 4: (key=A, v4)
Run 5: (key=A, v5)  ← 唯一有效

有效: 1份，实际存储: 5份
空间放大 = 5x
```

---

## 二、LSM-Tree 底层原理

### 核心结构

```
Write Path:
  Client → MemTable (内存) → Flush → L0 SSTable → Compaction → L1 → L2 → ... → Ln

Read Path:
  Client → MemTable → L0 → L1 → L2 → ... → Ln (逐层查找，合并结果)
```

设计哲学：**用顺序写替代随机写**。写入先进内存（MemTable），满后刷盘为有序文件（SSTable），通过后台 Compaction 合并。

### 写放大根因：Compaction

假设 size ratio = T（每层容量是上一层的 T 倍），层数 = L：
- **Leveled Compaction**：写放大 ≈ T × L（每次 compaction 可能重写整层）
- **Tiered Compaction**：写放大 ≈ L（每层只追加，不重写同层）

### 读放大根因：数据分散

- **Leveled Compaction**：读放大 ≈ L（每层最多查1个文件，同层 key 不重叠）
- **Tiered Compaction**：读放大 ≈ T × L（每层有 T 个 sorted run，key 重叠）

### 核心矛盾：跷跷板

| 策略 | 写放大 | 读放大 | 空间放大 |
|------|--------|--------|----------|
| Leveled Compaction | 高 (T×L) | 低 (L) | 低 |
| Tiered Compaction | 低 (L) | 高 (T×L) | 高 |
| FIFO (无 Compaction) | 最低 (1) | 最高 | 最高 |

**本质**：Compaction 做得越勤 → 数据越有序 → 读越快 → 但写放大越大。不可能三角。

### 通用优化手段

| 手段 | 优化目标 | 原理 |
|------|---------|------|
| Bloom Filter | 读放大 | 快速判断 key 不在某文件，跳过无效 IO |
| 分区/分桶 | 读+写 | 缩小 Compaction 和查询范围 |
| Tiered + Leveled 混合 | 平衡 | 上层 Tiered 减少写，下层 Leveled 减少读 |
| Compaction 调度优化 | 写放大 | 选择性 Compaction，避免不必要重写 |
| Z-order / Hilbert 排序 | 读放大 | 多维数据局部性更好，减少扫描文件数 |

---

## 三、技术演进链 — 站在巨人肩膀上

### 3.1 HBase 时代 — 发现问题

HBase 是第一代大规模 LSM-Tree 系统，**暴露了所有经典问题**。

#### HBase 的核心痛点

```
痛点1: Major Compaction 风暴
  → 定时触发，读写整个 Region 的全部 HFile
  → 凌晨跑时 CPU/IO 打满，影响在线服务

痛点2: 读放大严重
  → 一个 Region 5-10 个 HFile 很正常
  → 每次 Get 查所有 HFile
  → BlockCache 未命中时延迟飙升

痛点3: 空间放大 (HDFS 3副本)
  → 数据 → MemStore → HFile(1份) → HDFS(×3) = 3份
  → Compaction 期间新旧共存 → 短时间6份
```

#### HBase 的优化（不换系统）

**(1) 版本特性演进**：

| 版本 | 特性 | 解决什么 |
|-----|------|---------|
| 0.98 | Stripe Compaction | Region 分多个 stripe，Compaction 范围缩小 → 写放大降低 50%+ |
| 1.0 | Date-tiered Compaction | 时序数据按时间分层，旧数据不反复合并 → 写放大降低 |
| 2.0 | In-memory Compaction | MemStore 内部先合并再 Flush → Flush 次数减少 → 写放大降低 |
| 2.0 | MOB (Medium Object) | >100KB value 单独存储不参与 Compaction → 写放大大幅降低 |
| 2.0 | BucketCache (off-heap) | 堆外缓存，增大容量减少 GC → 读放大降低 |

**(2) 配置调优**：

```xml
<!-- 减少写放大: 提高 Flush 阈值 -->
<property>
  <name>hbase.hregion.memstore.flush.size</name>
  <value>268435456</value>  <!-- 256MB, 默认128MB -->
</property>

<!-- 减少写放大: 推迟 Major Compaction -->
<property>
  <name>hbase.hregion.majorcompaction</name>
  <value>604800000</value>  <!-- 7天, 默认1天 -->
</property>

<!-- 减少读放大: 启用 Bloom Filter -->
<property>
  <name>hbase.bloomfilter.type</name>
  <value>ROW</value>
</property>

<!-- 减少读放大: 控制 HFile 上限 -->
<property>
  <name>hbase.hstore.compactionThreshold</name>
  <value>3</value>  <!-- 3个HFile触发Minor Compaction -->
</property>
```

**(3) 任务调整**：

```bash
# 关闭自动 Major Compaction，改为手动调度到业务低峰
hbase.hregion.majorcompaction = 0

# 凌晨3点手动触发
0 3 * * * /usr/bin/hbase major_compact 'table_name'
```

**(4) 业务/数据调整**：

```
问题: 热点 rowkey 导致单 Region 写集中 → Compaction 压力大

反例 (热点):
  rowkey = "20260228_order_001"  → 同一天全在一个 Region

正例 (打散):
  rowkey = MD5("20260228_order_001")[0:4] + "_20260228_order_001"
  → 分散到多个 Region，每个 Compaction 压力降低
```

#### HBase 解决不了的根本问题

```
❌ 只支持 KV 点查和短 Scan，不支持复杂 SQL 分析
❌ HDFS 3副本是硬性要求，空间放大不低于 3x
❌ Compaction 和在线服务共享资源，无法真正隔离
❌ Schema 固定（列族），不适合灵活分析
```

→ 催生了数据湖表格式的出现

---

### 3.2 Iceberg 时代 — 换思路，文件级版本管理

Iceberg 看到 HBase 问题后换了思路：不在文件内部做 LSM 合并，而是在文件级别做版本管理。

#### Iceberg 解决了 HBase 的什么

```
✅ SQL 分析能力 (Parquet/ORC + Spark/Flink/Trino)
✅ 不强制 HDFS 3副本 (可用对象存储)
✅ Schema Evolution (加减列不影响已有数据)
✅ Time Travel (快照隔离)
✅ 分区演进 (不重写数据就能改分区)
```

#### Iceberg 自身的放大问题

**COW 写放大**：

```python
# Spark + Iceberg COW
spark.sql("""
    MERGE INTO orders t
    USING updates s ON t.order_id = s.order_id
    WHEN MATCHED THEN UPDATE SET t.status = s.status
""")

# updates 100条，分布在50个 Parquet 文件中
# → 50个文件每个: 读全部 → 改几行 → 写出新文件
# → 50 × 128MB = 6.4GB IO，有效变更 ~10KB
# 写放大 ≈ 65万倍
```

**MOR 读放大**：

```python
# Iceberg MOR (v2 position delete)
# 100次小批量更新后:
#   data files: 50个
#   delete files: 100个

# 查询时:
#   读 data file → 读所有关联 delete files → 逐行判断删除 → 合并
#   delete files 越多 → 读放大越严重
```

**流式小文件 = 空间 + 读放大**：

```
Flink → Iceberg, 每分钟 Checkpoint
  → 每分钟1个小文件 → 一天1440个
  → 小文件: Parquet 压缩效率低 → 空间放大
  → 文件多: 查询扫描1440个元数据 → 读放大
```

#### Iceberg 的优化

**(1) 版本特性**：

| 版本 | 特性 | 解决什么 |
|-----|------|---------|
| v2 (0.13+) | Position Delete (替代 Equality Delete) | 记行号而非全 key → 写放大降低 |
| 1.3+ | Object Storage Layout | 文件路径优化 → 读放大降低 |
| 1.4+ | MOR 优化 | delete file 与 data file 关联更高效 → 读放大降低 |

**(2) 配置调优**：

```sql
ALTER TABLE orders SET TBLPROPERTIES (
    'write.target-file-size-bytes' = '134217728',   -- 128MB 目标
    'write.distribution-mode' = 'hash',              -- 写前 hash 分发
    'commit.manifest.min-count-to-merge' = '10',     -- 10个 manifest 再合并
    'read.split.target-size' = '268435456'           -- 读 split 256MB
);
```

**(3) 任务调整 — 定期 Compaction**：

```python
# 合并小文件
spark.sql("""
    CALL catalog.system.rewrite_data_files(
        table => 'db.orders',
        strategy => 'sort',
        sort_order => 'order_id',
        options => map(
            'target-file-size-bytes', '134217728',
            'min-file-size-bytes', '67108864'
        )
    )
""")

# 合并 delete files (减少读放大)
spark.sql("""
    CALL catalog.system.rewrite_data_files(
        table => 'db.orders',
        options => map('delete-file-threshold', '3')
    )
""")

# 清理过期快照 (减少空间放大)
spark.sql("""
    CALL catalog.system.expire_snapshots(
        table => 'db.orders',
        older_than => TIMESTAMP '2026-02-21 00:00:00',
        retain_last => 5
    )
""")
```

**(4) 业务/数据调整**：

```
问题: 更新分散在大量文件中 → COW 写放大巨大

反例: 按 region 分区
  PARTITIONED BY (region)
  → 更新分散在所有 region → 每个分区文件都要重写

正例: 按日期分区
  PARTITIONED BY (days(create_time))
  → 更新集中在最近几天 → 只重写少量分区
  → 历史分区完全不受影响
```

#### Iceberg 解决不了的根本问题

```
❌ COW 写放大本质无法避免 (改1行必须重写整文件)
❌ MOR delete file 积累后读放大严重，依赖手动 Compaction
❌ 没有内置 LSM-Tree，更新密集场景效率远不如 KV 存储
❌ Compaction 依赖外部引擎调度，不是存储层自治
❌ 无原生 Changelog，CDC 场景需额外处理
```

→ 催生了 Paimon：把 LSM-Tree 嵌入数据湖表格式

---

### 3.3 Spark → Flink — 计算引擎的演进

在讲 Paimon 之前，先理清计算引擎的演进。

#### Spark 时代的问题

```
Spark Structured Streaming = 微批 (Micro-batch)
  → 每个微批是一个独立的批处理 Job
  → 每个 Job 有启动开销 (调度、资源申请)
  → 最小延迟 ~100ms，实际常在秒级

对数据湖写入的影响:
  → 每个微批产生独立的文件提交
  → 微批间隔越短 → 小文件越多 → 写放大、读放大
  → Spark 无法感知上下游 Changelog 语义 (+I/-U/+U/-D)
  → 所有更新都当"全量行"处理 → 存储层做更多无谓合并
```

#### Flink 解决了什么

```
✅ 真正的流处理 (事件驱动，非微批)
  → 数据到达即处理，延迟 ms 级
  → 写入更均匀，不会产生"批次性"小文件堆积

✅ 原生 Changelog 语义
  → Flink CDC 产生标准 Changelog (+I, -U, +U, -D)
  → 存储层可直接根据类型做增量合并
  → 不需要 full-row 对比 → 减少合并计算量

✅ Checkpoint 与 Snapshot 对齐
  → Flink Checkpoint = Paimon 原子提交
  → 没有额外 commit 开销
  → 间隔可控 (旋钮: 间隔短=实时性高+小文件多, 间隔长=反之)

✅ 常驻 Streaming Compaction
  → Spark: 合并小文件要启动新 Spark Job
  → Flink: 常驻 Streaming Job 持续做 Compaction
```

---

### 3.4 Paimon + Flink — 把 LSM 带回数据湖，但比 HBase 更聪明

#### Paimon 从前辈学到了什么

```
从 HBase:
  ✅ LSM-Tree 写入效率高 (顺序写 + 后台合并)
  ✅ Bloom Filter, 分层存储
  ❌ 不再绑定 HDFS 3副本
  ❌ 不再 KV 接口，而是 SQL 表

从 Iceberg:
  ✅ 列式存储 (ORC/Parquet) 做分析
  ✅ 快照隔离, Schema Evolution
  ✅ 分区裁剪
  ❌ 不再用 COW (改为 LSM append)
  ❌ 内置 Compaction (不完全依赖外部调度)

Paimon = LSM-Tree 引擎 + 数据湖表格式 + 列式存储
```

#### Paimon 和 Flink 到底各自做了什么 — 分层解决同一目标

**核心问题：他们说的是同一件事，还是各自解决了不同层的问题？**

**答案：不同层的问题，同一个最终目标。**

```
写放大的总量 = 计算层浪费 + 存储层 Compaction 开销

┌─────────────┬──────────────────────────────────────────────┐
│ Flink 解决  │ 计算层浪费:                                   │
│ (数据入口)  │  - 预 Shuffle → 减少跨 Bucket 写入            │
│             │  - 攒批 → 减少小文件                          │
│             │  - Changelog 语义 → 减少合并计算量             │
│             │                                              │
│             │ 效果: 让存储层收到的数据"更干净"               │
│             │       从源头减少 Compaction 工作量              │
├─────────────┼──────────────────────────────────────────────┤
│ Paimon 解决 │ 存储层 Compaction 开销:                       │
│ (数据存储)  │  - Universal Compaction → 选择性合并           │
│             │  - Lookup Compaction → 跳过无冲突数据          │
│             │  - Deletion Vector → 消除 MOR 读放大           │
│             │  - 异步 Compaction → 写入零等待                │
│             │                                              │
│             │ 效果: 即使同样的数据，Compaction 开销也更小     │
└─────────────┴──────────────────────────────────────────────┘

Flink = 减少问题的输入 (让数据更有序更干净)
Paimon = 优化问题的处理 (让 Compaction 更聪明)
两者叠加 → 端到端写放大最小化
```

#### Flink 在计算层的具体优化

```
1. 预 Shuffle (按 Bucket Key 分发)
   Flink write 算子之前，按 bucket-key 做 keyBy
   → 同一 key 的所有变更落到同一 Bucket
   → Paimon LSM 不需要跨 Bucket 合并
   → 没有 Flink 预分发？每个 Bucket 收到任意 key → 合并范围爆炸

2. 攒批写入 (Write Buffer)
   Flink Paimon Sink 内存中维护 write-buffer-size 缓冲区
   → 攒够再 Flush，不逐条写文件
   → 减少小文件 → 减少 Compaction 合并对象数

3. Changelog 规范化
   Flink CDC 产生标准 Changelog (+I, -U, +U, -D)
   → Paimon 根据类型做增量合并
   → 不需要 full-row 对比判断插入/更新
   → 减少合并计算开销 (间接减少写放大)

4. Checkpoint = Snapshot
   Flink Checkpoint 完成 = Paimon 原子提交 Snapshot
   → 无额外 commit 开销
   → Checkpoint 间隔 = Snapshot 间隔 (可调旋钮)
```

#### Paimon 在存储层的具体优化

```
1. Universal Compaction (从 RocksDB 借鉴改良)
   HBase: 只有 Size-tiered 和 Leveled
   RocksDB: 引入 Universal Compaction
   Paimon: 基于 RocksDB Universal，针对数据湖改良

   原理: 不强制每层 key 不重叠 (Leveled 约束太强 → 写放大大)
         也不放任重叠 (Tiered → 读放大大)
         动态评估: "合并这几个 sorted run 的收益 > 成本？"
         → 选择性合并

   效果: 写放大比 Leveled 低 50%+，读放大比 Tiered 低

2. Lookup Compaction (Paimon 独创)
   传统: 合并时读取所有参与的 SST，排序后写出
   Lookup: 合并时只查找有冲突的 key (通过索引定位)
   → 新 SST 与旧 SST 无交集 → 跳过，不重写旧 SST
   → 大幅减少无谓重写

3. Deletion Vector (Paimon 0.8+, 借鉴 Delta Lake 3.0)
   Iceberg MOR: delete file 积累 → 读时合并所有 delete file → 读放大
   Paimon DV: Bitmap 标记在 data file 元数据中
   → 读时直接跳过标记行，不读额外 delete file
   → 读放大从 O(N个delete file) 降到 O(1)
   → 直接解决 Iceberg MOR 核心缺陷

4. 异步 Compaction (架构设计 + Flink 执行)
   HBase: Compaction 和 RegionServer 同进程，争 CPU/IO
   Iceberg: Compaction 是手动触发的批任务
   Paimon: Compaction 可以是独立 Flink Streaming Job

   -- 写入 (持续运行)
   INSERT INTO paimon_table SELECT * FROM kafka_source;

   -- 独立 Compaction (持续运行, 独立资源)
   CALL sys.compact('db.paimon_table');

   写入不等 Compaction → 写入链路写放大 = 0
   Compaction 独立资源 → 不影响在线查询
```

#### Paimon + Flink 的四类优化

**(1) 版本特性**：

| 组件 | 特性 | 解决什么 |
|-----|------|---------|
| Paimon 0.4+ | Universal Compaction | 动态平衡读写放大 |
| Paimon 0.5+ | Lookup Compaction | 跳过无冲突数据 → 写放大降低 |
| Paimon 0.8+ | Deletion Vector | Bitmap 标记替代 delete file → 读放大 O(1) |
| Flink 1.16+ | Paimon Connector 深度集成 | 预 Shuffle + 攒批 → 源头减少写放大 |

**(2) 配置调优**：

```sql
-- Paimon 表配置
CREATE TABLE orders (
    order_id BIGINT,
    status STRING,
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'write-buffer-size' = '256mb',                -- 攒批大小 (越大→flush越少→写放大低)
    'changelog-producer' = 'lookup',              -- changelog 模式
    'compaction.max-sorted-run-num' = '5',        -- 最大 sorted run 数 (越小→读放大低→写放大高)
    'deletion-vectors.enabled' = 'true',          -- 开启 Deletion Vector
    'num-sorted-run.stop-trigger' = '10'          -- sorted run 超过此值暂停写入等 Compaction
);

-- Flink 侧配置
SET 'execution.checkpointing.interval' = '2min';  -- Checkpoint 间隔
-- 短 → 小文件多 → 写放大大
-- 长 → 延迟高 → 但写放大小
```

**(3) 任务/算子调整**：

```sql
-- 独立 Compaction Job (与写入 Job 隔离资源)
-- 写入 Job 并行度 = 数据量决定
-- Compaction Job 并行度 = 磁盘 IO 能力决定

-- 写入 Job
INSERT INTO paimon_table SELECT * FROM source;

-- 独立 Compaction Job (另一个 Flink 作业)
CALL sys.compact('db.paimon_table');

-- 或在 Flink SQL 中配置自动触发
SET 'table.exec.compact.enabled' = 'true';
```

**(4) 业务/数据调整**：

```sql
-- Bucket Key 选择: 高基数列，让数据均匀分布
-- 反例: 低基数 bucket-key
WITH ('bucket-key' = 'status')  -- status 只有几个值 → 数据倾斜 → 单 bucket 压力大

-- 正例: 高基数 bucket-key
WITH ('bucket-key' = 'order_id')  -- 均匀分布

-- Partial Update: 只写变化列，减少写入量
WITH ('merge-engine' = 'partial-update')
-- 只传 (pk, changed_column) 而非全部列
-- → 写入量减少 → Compaction 合并量减少 → 写放大降低
```

---

### 3.5 Doris — 另一条路：OLAP 引擎内置 LSM

Doris 是一体化 OLAP，不依赖外部存储和计算引擎。

#### Doris 从前辈学到了什么

```
从 HBase:
  ✅ LSM-Tree 分层结构
  ❌ 面向分析 (列式存储)，不是 KV 点查

从 Iceberg 的教训:
  ✅ 不走 COW，内置 Compaction
  ✅ 分区裁剪、列裁剪

独创:
  Merge-on-Write (2.0+) → 根本性解决 Unique Key 读放大
```

#### Doris 的放大问题与优化

**(1) 版本特性 — Merge-on-Write（最重要）**：

```
Doris 1.x (Merge-on-Read):
  写入 → Rowset 1, 2, ... N (各自可能含同一 key)
  查询 → 读所有 Rowset → 运行时合并去重 → 返回
  → 读放大严重

Doris 2.0 (Merge-on-Write):
  写入 → 先查主键索引是否存在该 key
       → 存在则标记旧行 delete (Bitmap)
       → 写新行到新 Rowset
  查询 → 每个 Rowset 已去重，直接读
  → 读放大 O(N) → O(1)
  → 代价: 写入多一次查找

  本质: 把读放大成本转移到写入
        "写少读多"的分析场景，这是正确的 trade-off
```

**(2) 配置调优**：

```sql
CREATE TABLE orders (
    order_id BIGINT,
    status VARCHAR(20),
    amount DECIMAL(10,2)
)
UNIQUE KEY(order_id)
DISTRIBUTED BY HASH(order_id) BUCKETS 16
PROPERTIES (
    "enable_unique_key_merge_on_write" = "true",  -- 开启 MOW
    "compaction_policy" = "time_series",           -- 时序优化
    "disable_auto_compaction" = "false"
);

-- BE 配置
-- cumulative_compaction_num_singleton_deltas = 5
-- base_compaction_interval_seconds_since_last_operation = 86400
-- compaction_task_num_per_disk = 4
```

**(3) 任务调整**：

```bash
# 手动触发 Compaction (业务低峰)
curl -X POST http://be_host:8040/api/compaction/run?tablet_id=xxx&compact_type=cumulative

# 合理分桶 → 每个桶独立 Compaction
# 反例: BUCKETS 1 → 全数据一起 Compaction
# 正例: BUCKETS 16 → 各自独立，范围小
```

**(4) 业务/数据调整**：

```sql
-- 2.1+ Partial Update: 只写变化列
-- 反例: 全量列写入
INSERT INTO t VALUES (1, 'new_status', NULL, NULL, NULL, ...);

-- 正例: 只写变化列
-- 设置 partial_columns 只传 (pk, changed_col)
-- → 存储层合并量减少 → 写放大降低

-- 数据模型选择
-- Duplicate: 无去重 → 写放大最低，但不支持更新
-- Aggregate: 预聚合 → Compaction 时直接聚合，减少数据量
-- Unique + MOW: 去重 → 读放大最低
```

---

## 四、完整演进总结

### 技术演进链

```
HBase (2008) → Iceberg (2018) → Doris 2.0 (2023) → Paimon (2023) → Flink+Paimon 深度集成 (2024)
                Spark (2014) ──────────────────────→ Flink (2019 成熟)

HBase 时代的问题:
  ├── KV 模型无法 SQL 分析 ──────→ Iceberg: 列式存储 + SQL ✅
  ├── HDFS 3副本空间放大 ────────→ Iceberg: 对象存储 ✅
  ├── Major Compaction 风暴 ─────→ Iceberg: 没 LSM 没风暴 ✅  (但引入 COW 写放大 ❌)
  └── Compaction 与在线抢资源 ───→ Paimon: 异步 Compaction Job ✅

Iceberg 时代的问题:
  ├── COW 写放大巨大 ───────────→ Paimon: LSM append 不重写 ✅
  ├── MOR delete file 读放大 ───→ Paimon: Deletion Vector (Bitmap) ✅
  ├── Compaction 依赖手动调度 ──→ Paimon: 内置触发 + Flink 异步 Job ✅
  ├── 无原生 Changelog ────────→ Paimon: 原生 Changelog ✅
  └── 无原生流写入优化 ────────→ Flink: 预 Shuffle + 攒批 + Changelog ✅

Spark 时代的问题:
  ├── 微批延迟高 ──────────────→ Flink: 真正流处理 ✅
  ├── 批 Compaction 启动新 Job →  Flink: 常驻 Streaming Compaction ✅
  └── 无 Changelog 语义 ──────→ Flink CDC: 端到端 Changelog ✅
```

### 四种优化手段对照表

| 手段 | HBase | Iceberg | Paimon+Flink | Doris |
|------|-------|---------|-------------|-------|
| **版本特性** | In-memory Compaction, MOB, Date-tiered | Position Delete, MOR v2 | Deletion Vector, Lookup Compaction, Universal Compaction | Merge-on-Write, Partial Update |
| **配置调优** | flush.size, majorcompaction 间隔, bloom filter | target-file-size, distribution-mode | write-buffer-size, compaction.max-sorted-run-num, checkpoint 间隔 | compaction_policy, buckets 数, MOW 开关 |
| **任务/算子** | 手动 Major Compaction, 预分区 | rewrite_data_files, expire_snapshots | 独立 Compaction Job, Flink 并行度 | 手动 Compaction, 分桶策略 |
| **业务/数据** | rowkey 加盐/反转, 列族拆分 | 按更新热度分区, Sort Order | Bucket Key 选高基数列, Partial Update | 模型选择 (Unique/Aggregate/Duplicate) |

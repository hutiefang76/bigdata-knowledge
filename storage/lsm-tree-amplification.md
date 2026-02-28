# LSM-Tree：为什么大数据组件都选择它，以及它要付出什么代价

---

## 一、LSM-Tree 是什么，为什么那么多大数据组件都用它

### 1.1 传统数据库的瓶颈：磁盘最怕随机写

```
机械硬盘 (HDD):                         固态硬盘 (SSD):
  顺序写: ~200 MB/s                        顺序写: ~3000 MB/s
  随机写: ~1 MB/s  ← 差 200 倍             随机写: ~200 MB/s (4K小块)  ← 差 ~15 倍, 且加速磨损
```

MySQL InnoDB、PostgreSQL 使用 B+Tree：

```
B+Tree 写入一条记录:
  1. 读取目标 Page (16KB)  ← 随机读
  2. 修改 Page 中几十字节
  3. 写回整个 Page          ← 随机写
  4. 可能触发页分裂 → 再写几个 Page

每次写入 = 多次随机 IO
写入量达到每秒数万~数十万条时，B+Tree 随机 IO 成为瓶颈
```

这在传统 OLTP（每秒几千 TPS）下不是问题，但在大数据场景——日志、IoT 传感器、CDC 变更流、时序数据——**每秒数十万甚至百万条写入**时，B+Tree 撑不住。

### 1.2 论文起源

**1996 年**，Patrick O'Neil、Edward Cheng、Dieter Gawlick、Elizabeth O'Neil 在 Acta Informatica 发表论文：

> *"The Log-Structured Merge-Tree (LSM-Tree)"*

O'Neil 来自 **University of Massachusetts Boston**，核心洞察：

> 既然磁盘的顺序写比随机写快 100~200 倍，那为什么不把所有写入都变成顺序写？代价是后台需要合并（Compaction），但这个合并也是顺序读写，总体吞吐远高于 B+Tree。

论文在发表后相当长时间内只是学术成果，直到互联网规模爆发才被工业界大规模采用。

### 1.3 谁第一个实践？

```
时间线:
  1996  论文发表 (O'Neil et al.)
          │
          │  (沉寂 ~10 年，学术界讨论，工业界未大规模采用)
          │
  2006  Google Bigtable 论文
          │  内部使用 GFS + Tablet Server (MemTable → SSTable → Compaction)
          │  这是 LSM-Tree 思想的第一个超大规模工业实践
          │  但 Bigtable 是 Google 内部系统，未开源
          │
  2007  Apache HBase
          │  Bigtable 的开源复刻，首版随 Hadoop 0.15 发布 (2007-10)
          │  第一个让普通公司也能用上 LSM-Tree 的系统
          │  暴露了 LSM 的所有经典问题 (Compaction 风暴、读放大、HDFS 空间放大)
          │
  2011  Google LevelDB
          │  Jeff Dean 和 Sanjay Ghemawat 编写的精简 LSM 实现
          │  ~20000 行 C++ 代码，定义了 MemTable→SSTable→Compaction 标准范式
          │  单线程，教科书级别，几乎所有后续实现的起点
          │
  2012  Facebook RocksDB
          │  Fork LevelDB，加入多线程 Compaction、Column Family、Universal Compaction
          │  成为工业界事实标准，被 TiKV、Pegasus、FoundationDB 等直接使用
          │
  2012+ 爆发式扩散
       ├── Cassandra (2008 Facebook 开源 → 2009 Apache 孵化)
       ├── ScyllaDB (C++ 重写 Cassandra)
       ├── TiKV/TiDB (PingCAP, RocksDB 深度定制)
       ├── OceanBase (蚂蚁集团, 自研 LSM)
       ├── CockroachDB/Pebble (Go 重写 LSM)
       ├── Doris/StarRocks (列式 LSM OLAP)
       ├── X-Engine (阿里巴巴, FPGA 加速 LSM)
       ├── TerarkDB (Terark, 后被字节跳动收购, 压缩上直接搜索)
       └── Paimon (数据湖格式内置 LSM)
```

**真实历史的关键点**：
- Bigtable（2006）是 LSM-Tree 从论文到工业的转折点，但它不开源
- HBase（2007）让 LSM-Tree 普及，但也第一个大规模暴露了放大问题
- LevelDB（2011）不是第一个实践者，而是第一个干净的参考实现
- RocksDB（2012）才是今天工业界的事实标准

### 1.4 为什么这些组件都选择 LSM-Tree

| 指标 | B+Tree (InnoDB) | LSM-Tree (RocksDB) | 倍数 |
|------|----------------|-------------------|------|
| 随机写吞吐 | 数千~万 TPS | 数万~十万 TPS | **4~10x** |
| 顺序写吞吐 | ~20,000 TPS | ~200,000+ TPS | **~10x** |
| 点查延迟 | O(log N) 稳定 | O(L) L=层数，有波动 | B+Tree 略优 |
| 范围查询 | 快（叶节点有序链表） | 需合并多层（较慢） | B+Tree 优 |
| 空间利用率 | ~50-70% (页分裂碎片) | ~80-95% (Compaction 后紧凑) | LSM 优 |

> 注: 绝对 TPS 高度依赖硬件配置与数据集大小, 此处为数量级对比。比值经 Percona、smalldatum 等独立测试验证。

**结论**：LSM-Tree 用后台 Compaction 的代价，换来数倍以上写入吞吐。大数据场景几乎都是**写密集型**（日志采集、IoT 上报、CDC 同步、时序监控），这个 trade-off 是值得的。

### 1.5 使用 LSM-Tree 的主流组件一览

> 详细实现对比见第二篇：[lsm-tree-implementations-comparison.md](./lsm-tree-implementations-comparison.md)

| 类别 | 组件 | LSM 来源 |
|------|------|---------|
| 参考实现 | LevelDB, RocksDB | 原创 / LevelDB 演进 |
| NoSQL | HBase, Cassandra, ScyllaDB | 自研 |
| NewSQL | TiKV, OceanBase, CockroachDB/Pebble | RocksDB定制 / 自研 / Go自研 |
| OLAP | Doris, StarRocks | 自研列式 LSM |
| 数据湖 | **Paimon** (唯一内置 LSM 的湖格式) | 自研 |
| 流引擎 State | Flink (RocksDB StateBackend) | RocksDB |
| 专用引擎 | BadgerDB, X-Engine, TerarkDB | WiscKey / 自研 / Terark→字节跳动 RocksDB fork |

---

## 二、LSM-Tree 底层原理

### 2.1 核心结构

```
Write Path:
  Client → WAL (磁盘, 追加写, 用于崩溃恢复)
         → MemTable (内存, SkipList/红黑树, 有序)
                ↓ 满了 (如 64MB)
           Flush → L0 SSTable (Sorted String Table, 不可变有序文件)
                        ↓ L0 文件太多
                   Compaction → L1 → L2 → ... → Ln

Read Path:
  Client → MemTable → L0 → L1 → L2 → ... → Ln
           (逐层查找, 找到最新版本即停)
```

### 2.2 完整写入流程

```
Step 1: WAL + MemTable
  ┌──────────────────────┐
  │  MemTable (sorted)   │  ← 纯内存操作, 红黑树或跳表
  │  key=A → v3          │
  │  key=B → v1          │
  │  key=C → v2          │
  └──────────┬───────────┘
             │ 满了
             ↓
Step 2: Flush → SSTable
  ┌──────────────────────┐
  │  L0/sst_001          │  ← 一次顺序写
  │  [A:v3, B:v1, C:v2]  │
  └──────────────────────┘
  ┌──────────────────────┐
  │  L0/sst_002          │  ← 又一次 Flush
  │  [A:v5, D:v1, E:v1]  │     A 出现了两次 (v3 和 v5)
  └──────────────────────┘
             │ L0 文件太多
             ↓
Step 3: Compaction
  读取 sst_001 + sst_002
  → 合并排序 → A 取最新 v5
  → 写出新文件 sst_003 到 L1
  → 删除旧文件

  同样的数据被写了两次 → 这就是写放大的来源
```

### 2.3 两大经典 Compaction 策略

#### Leveled Compaction（读优先）

```
L0: [sst_1] [sst_2] [sst_3]     ← key 范围可重叠
L1: [sst_a] [sst_b] [sst_c]     ← 同层 key 不重叠！
L2: [sst_x] [sst_y] [sst_z]     ← 同层 key 不重叠！

Compaction: L0 文件与 L1 所有重叠文件合并 → 可能重写 L1 多个文件
→ 写放大高, 读放大低
代表: LevelDB, RocksDB 默认, Pebble, HBase Minor
```

#### Tiered Compaction（写优先）

```
L0: [run_1] [run_2] [run_3]     ← 同层允许多个 sorted run
L1: [run_4] [run_5]             ← key 范围可重叠！

Compaction: 同层积累 T 个 run 后合并为 1 个推到下层 → 每个 run 只被写一次
→ 写放大低, 读放大高
代表: Cassandra STCS, ScyllaDB STCS
```

#### 第三条路：Universal Compaction（动态平衡）

```
不强制每层不重叠 (Leveled 太激进)
也不放任全部重叠 (Tiered 太懒)
动态评估: "合并这几个 sorted run 的收益 > 成本？" → 选择性合并

代表: RocksDB Universal, Paimon Universal
```

---

## 三、LSM-Tree 的代价：三种放大问题

放大不是 LSM-Tree 的缺陷，而是用顺序写换取高吞吐的**必然代价**。但如何最小化这个代价，是过去 20 年所有 LSM 组件努力的方向。

### 3.1 写放大 (Write Amplification)

**定义**：一条数据从写入到最终稳定，实际磁盘写入量 / 用户原始写入量。

**根因**：Compaction 反复读出旧数据、合并、写出新文件。

```
用户写入 1 条 → MemTable → Flush L0 (写1次)
                              → Compaction L0→L1 (写第2次)
                              → Compaction L1→L2 (写第3次)
                              → ...

Leveled: 写放大 ≈ T × L (T=层间大小比, L=层数), 典型 10~30x
Tiered:  写放大 ≈ L, 典型 2~7x
```

**具体例子 — HBase 高频更新同一行**：

```java
for (int i = 0; i < 10000; i++) {
    Put put = new Put(Bytes.toBytes("row_001"));
    put.addColumn(CF, COL, Bytes.toBytes("value_" + i));
    table.put(put);
}
// 10000次写入 → MemStore → 多次 Flush → 多个 HFile
// Major Compaction 全部合并, 只保留最新1条
// 磁盘实际写: 10000次 Flush + N次 Compaction 重写
// 有效数据: 1行
```

### 3.2 读放大 (Read Amplification)

**定义**：读取一条记录时，实际磁盘读取量 / 该记录数据量。

**根因**：数据分散在多层多个文件中，查询需逐层查找。

```
Compaction 没跟上时:
  L0: 20个文件 (key 范围全部重叠!)
  L1: 30个文件
  L2: 49个文件

  查 1 条: → L0 的 20 个文件逐个打开 → Bloom Filter 判断 → L1 查 1 个 → L2 查 1 个
  为了 1 行数据, 打开 20+ 个文件

Compaction 充分时:
  L0: 空
  L1: 10个 (key 不重叠)

  查 1 条: → 二分定位 1 个文件 → Bloom Filter → Data Block → 返回
```

### 3.3 空间放大 (Space Amplification)

**定义**：磁盘实际占用 / 有效数据量。

**根因**：旧版本和 tombstone 在 Compaction 前一直占用空间。

```
同一 key 更新 5 次 (Tiered Compaction):
  Run 1: (key=A, v1)  ← 旧
  Run 5: (key=A, v5)  ← 唯一有效
  有效 1 份, 实际存储 5 份 → 空间放大 5x

DELETE 不是真删除:
  写入 tombstone 标记 → 原始数据仍在 → 等 Compaction 清理
```

### 3.4 不可能三角

```
        写放大 低
           /\
          /  \
         / 不可能 \
        /  三角    \
       /____________\
  读放大 低      空间放大 低

Leveled:   读低 + 空间低 → 写高
Tiered:    写低           → 读高 + 空间高
Universal: 在三角内动态找最优点
```

**这就是为什么所有 LSM 组件都在这个三角内做优化，而不是消除放大——放大是高吞吐写入的本质代价。**

---

## 四、解决放大的手段体系

以 LSM-Tree 的三个问题为主线，梳理各类解决方案。

### 4.1 降低写放大

| 手段 | 原理 | 代表组件 |
|------|------|---------|
| **Universal Compaction** | 选择性合并，只在收益>成本时触发 | RocksDB, Paimon |
| **UCS 统一策略** | 单参数 W 连续调节写/读平衡 | Cassandra 5.0 |
| **KV 分离** | 大 Value 存独立文件，Compaction 只移动 Key+指针 | BadgerDB(WiscKey), TiKV(Titan), Pebble(ValueSep) |
| **Macro Block 增量合并** | 只重写脏块，干净块引用复用 | OceanBase (2MB Macro Block) |
| **异步 Compaction** | 写入与合并隔离到独立资源池 | Paimon(独立Flink Job), OceanBase(轮转合并) |
| **In-Memory Compaction** | MemTable 内先合并再 Flush → Flush 次数减少 | HBase 2.0+ |
| **MOB** | 大 Value 不参与 Compaction | HBase 2.0+ |
| **FPGA 加速** | Compaction 执行卸载到硬件 | X-Engine |
| **Lookup Compaction** | 只合并有 key 冲突的文件，无交集则跳过 | Paimon |
| **Flink 预 Shuffle + 攒批** | 数据写入前按 Bucket Key 分发+攒批 → 源头减少合并量 | Flink + Paimon |
| **SST Compaction Guard** | SST 边界对齐 Region 边界 → 避免跨 Region 重写 | TiKV |

### 4.2 降低读放大

| 手段 | 原理 | 代表组件 |
|------|------|---------|
| **Bloom Filter** | 快速判断 key 不在某文件，跳过无效 IO | 所有 LSM 组件 |
| **Merge-on-Write (MoW)** | 写入时即去重，查询无需合并多版本 | Doris 2.1+, StarRocks, Paimon MOW |
| **Deletion Vector** | Bitmap 标记已删行，读时直接跳过 | Paimon 0.8+, Doris |
| **L0 Sublevel** | L0 内分虚拟子层，支持并发 Compaction | Pebble (CockroachDB) |
| **分区/分桶裁剪** | 缩小查询扫描范围 | 所有 LSM 组件 |
| **Zone Map (min/max 统计)** | 列级别统计，跳过无关数据块 | Doris, StarRocks |
| **HyperClockCache** | 无锁并发缓存，高负载下延迟更稳定 | RocksDB 10.7+ |
| **CO-Index** | 在压缩数据上直接搜索，无需解压 | TerarkDB |

### 4.3 降低空间放大

| 手段 | 原理 | 代表组件 |
|------|------|---------|
| **及时 Compaction** | 合并清理旧版本和 tombstone | 所有 LSM 组件 |
| **ICS 增量策略** | SSTable 分段，Major Compaction 空间开销从 50% 降到 5% | ScyllaDB |
| **渐进合并** | DDL 变更后分 N 次逐步重写 | OceanBase |
| **ZSTD 压缩** | 高压缩比列式存储 | X-Engine (1/3~1/10), OceanBase |
| **MVCC GC Filter** | Compaction 底层自动清理旧 MVCC 版本 | TiKV |
| **Periodic Full Compaction** | 空闲时全量合并清除冗余 | TiKV v7.6+ |

---

## 五、各组件的 LSM-Tree 具体实践

### 5.1 HBase — 第一个大规模暴露 LSM 问题的系统

**版本**: 2.6.4

```
架构:
  Client → RegionServer → MemStore (per Column Family)
                                ↓ Flush
                           HFile (SSTable on HDFS)
                                ↓ Compaction
                           合并后的 HFile
```

#### HBase 暴露的问题

```
问题1: Major Compaction 风暴
  定时触发, 读写整个 Region 全部 HFile → CPU/IO 打满, 影响在线服务

问题2: 读放大
  一个 Region 5-10 个 HFile 很正常 → 每次 Get 查所有 HFile

问题3: 空间放大
  HDFS 3 副本 → 数据写 1 份, 存 3 份 → Compaction 期间新旧共存 → 短时间 6 份
```

#### HBase 怎么解决

**(1) 版本特性**:

| 版本 | 特性 | 解决什么 |
|-----|------|---------|
| 0.98 | Stripe Compaction | Region 分条带，缩小 Compaction 范围 → 写放大 -50% |
| 1.0 | Date-tiered Compaction | 时序数据旧数据不反复合并 → 写放大降低 |
| 2.0 | In-memory Compaction | MemStore 内先合并再 Flush → 写放大降低 |
| 2.0 | MOB | >100KB value 独立存储不参与 Compaction → 写放大大幅降低 |
| 2.0 | BucketCache (off-heap) | 堆外缓存 → 读放大降低 |

**(2) 配置调优**:

```xml
<!-- 减少写放大: 提高 Flush 阈值 -->
<property>
  <name>hbase.hregion.memstore.flush.size</name>
  <value>268435456</value>  <!-- 256MB, 默认128MB -->
</property>

<!-- 减少写放大: 关闭自动 Major Compaction, 手动调度到低峰 -->
<property>
  <name>hbase.hregion.majorcompaction</name>
  <value>0</value>  <!-- 0=关闭自动 Major Compaction; HBase 2.x 默认7天 -->
</property>

<!-- 减少读放大: Bloom Filter -->
<property>
  <name>hbase.bloomfilter.type</name>
  <value>ROW</value>
</property>

<!-- 减少读放大: HFile 数量上限 -->
<property>
  <name>hbase.hstore.compactionThreshold</name>
  <value>3</value>
</property>
```

**(3) 任务调整**:

```bash
# 关闭自动 Major Compaction, 手动调度到低峰
hbase.hregion.majorcompaction = 0
0 3 * * * /usr/bin/hbase major_compact 'table_name'
```

**(4) 业务调整**:

```
问题: 热点 rowkey → 单 Region 写集中 → Compaction 压力大
解决: rowkey 加盐分散
  反例: "20260228_order_001"         → 同天全在一个 Region
  正例: MD5(key)[0:4] + "_" + key   → 分散到多个 Region
```

#### HBase 无法解决的根本问题

- 只支持 KV 点查, 不支持 SQL 分析
- HDFS 3 副本, 空间放大下限 3x
- Compaction 与在线服务共享资源
- Schema 固定 (列族)

---

### 5.2 Cassandra / ScyllaDB — 分布式 LSM 的 Compaction 进化

#### Cassandra 5.0: UCS 统一策略

**版本**: 5.0.6

Cassandra 过去有 STCS (写优)、LCS (读优)、TWCS (时序) 三种策略，切换需要全量 Compaction。5.0 引入 **UCS (Unified Compaction Strategy)**，用一个参数 W 连续调节：

```
W < 0 → Leveled 行为 (高写放大, 低读放大)
W = 0 → 平衡点
W > 0 → Tiered 行为 (低写放大, 高读放大)

不同层可设不同 W → 如 L0 用 Tiered, 深层用 Leveled
参数热切换, 无需全量 Compaction
```

#### ScyllaDB: ICS 增量策略

**版本**: 2025.4.3

ScyllaDB (C++ 重写 Cassandra) 独创 **ICS (Incremental Compaction Strategy)**:
- 大 SSTable 拆为 1GB 的 Run 段
- Major Compaction 空间开销从 **50% 降到 5%**
- 支持跨层 tombstone 清理

Shard-per-core 架构: 每个 CPU 核心独立内存/存储/网络, 消除线程竞争, Compaction 也是 per-shard 执行。

---

### 5.3 TiKV — RocksDB 深度定制 + Titan KV 分离

**版本**: v8.5.5 LTS

TiKV 运行两个 RocksDB 实例 (`raftdb` + `kvdb`)，在 RocksDB 之上做了三大定制：

| 优化 | 原理 | 效果 |
|------|------|------|
| **SST Compaction Guard** | SST 边界对齐 Region 边界 | 写放大显著降低 |
| **Flow Control** | 替换 Write Stall 为分层限流 | 写入吞吐更平滑 |
| **MVCC GC Filter** | Compaction 底层自动清理旧版本 | 空间放大降低 |

**Titan KV 分离 (v7.6+ 新集群默认启用)**:

```
Value >= 32KB (v7.6+ 默认阈值, 旧版本默认 1KB) → 分离到 Blob File, LSM 只存指针
Compaction 只移动 Key + 指针 (几十字节), 不移动大 Value
→ Value=1KB: QPS 提升 2x
→ Value=32KB: QPS 提升 6x
→ 代价: 磁盘空间增大, 大范围 Scan 变慢
```

---

### 5.4 OceanBase — 自研 LSM, Macro Block 增量合并

**版本**: V4.3.5 / V4.2.5

OceanBase 的 LSM-Tree 与 RocksDB 系有根本差异：

```
RocksDB: Compaction 单位 = 整个 SSTable (数十~数百 MB)
OceanBase: Compaction 单位 = 2MB Macro Block
  → 只重写被修改的 Macro Block, 未修改的直接引用复用
  → 写放大远低于 RocksDB
```

**三级 Compaction**: Mini SSTable → Minor SSTable → Major SSTable (每日低峰)

**轮转合并**: 利用多副本架构, Compaction 期间查询路由到其他副本 → **彻底隔离 Compaction 对在线查询的影响**。这是单机 LSM (RocksDB/HBase) 做不到的。

---

### 5.5 CockroachDB / Pebble — Go 原生 LSM + Value Separation

**版本**: v26.1.0

Pebble 是 Go 重写的 LSM 引擎, 核心创新:

- **L0 Sublevel**: L0 分虚拟子层, 支持并发 Compaction → 减少写入堆积
- **Value Separation (v25.4 GA)**: 大 Value 存入独立 Blob File → 宽行写密集场景吞吐提升 **50-60%**

与 TiKV Titan、BadgerDB WiscKey 一脉相承，但 Pebble 是纯 Go 实现, 无 CGO。

---

### 5.6 Doris / StarRocks — OLAP 引擎内置列式 LSM

#### Doris

**版本**: 4.0.3

```
Doris 1.x (MoR): 查询时合并多 Rowset → 读放大严重
Doris 2.1+ (MoW): 写入时查主键索引 → Delete Bitmap 标记旧行 → 查询直接读
  → 查询性能提升 10x
  → 代价: 写入多一次查找
```

Compaction: Cumulative (小 Rowset 合并) + Base (合并到基线) + Segment (Rowset 内部)。

#### StarRocks

**版本**: v4.0.6

Primary Key 表 = LSM + MoW, 通过内存主键索引 + **DelVector** (Bitmap) 实现写入时去重。

v4.0 云原生优化: Smarter Compaction 减少 70-90% 云存储 API 调用。

---

### 5.7 Paimon + Flink — 把 LSM 带回数据湖

**版本**: Paimon 1.3.x

Paimon 是**唯一在数据湖表格式层面内置 LSM-Tree** 的系统。

```
Table → Partition → Bucket → LSM-Tree (每个 Bucket 独立)
```

#### 三种表模式

| 模式 | 写放大 | 读放大 | 原理 |
|------|--------|--------|------|
| **MOR** (默认) | 低 | 高 | 只做 minor compaction, 读时合并 |
| **COW** | 高 | 无 | 每次写入触发全量合并 |
| **MOW** | 中 | **极低** | LSM 点查 → Deletion Vector (Bitmap) → 读时过滤 |

#### Flink 和 Paimon 各自解决什么

```
写放大总量 = 计算层浪费 + 存储层 Compaction

Flink (计算层):
  - 预 Shuffle 按 Bucket Key 分发 → 减少跨 Bucket 写入
  - 攒批 (write-buffer-size) → 减少小文件
  - Changelog 语义 (+I/-U/+U/-D) → 减少合并计算量
  → 让 Paimon 收到的数据"更干净", 从源头减少 Compaction 工作量

Paimon (存储层):
  - Universal Compaction → 选择性合并
  - Lookup Compaction → 跳过无冲突数据
  - Deletion Vector → 读放大 O(1)
  - 异步 Compaction (独立 Flink Job) → 写入零等待
  → 即使同样的数据, Compaction 开销也更小

两者叠加 → 端到端写放大最小化
```

#### 其他数据湖格式的放大问题

Paimon 之外的三大湖格式 (Iceberg / Delta Lake / Hudi) **不使用 LSM-Tree**，但同样面临放大问题：

| 湖格式 | 架构 | 写放大 | 读放大 | 怎么解决 |
|--------|------|--------|--------|---------|
| **Iceberg** | 快照 + COW/MOR | COW: 改1行重写整文件 | MOR: delete file 积累 | V3 Deletion Vector (Bitmap); 手动 rewrite_data_files |
| **Delta Lake** | COW + Deletion Vector | COW 重写; DV 低 | DV 累积后退化 | OPTIMIZE 物理清理; Liquid Clustering |
| **Hudi** | Base + Log (类 LSM) | MOR 低; COW 高 | MOR 需合并 Log | LogCompaction (Log 间合并不写 Base); LSM Timeline |

其中 **Hudi 的 MOR 在概念上最接近 LSM** (Base File ≈ 深层 Level, Log File ≈ L0)，但它的 Compaction 粒度和策略远不如 Paimon 精细。

---

### 5.8 专用优化引擎

#### BadgerDB (Dgraph) — WiscKey KV 分离

**版本**: v4.x

LSM-Tree 只存 Key + Value 指针, 大 Value 存独立 vLog。Compaction 只重写 Key+指针 → **写放大接近 1x**。代价: 随机读 vLog + vLog GC 开销。纯 Go, 无 CGO。

#### PolarDB X-Engine (阿里巴巴) — FPGA 加速 Compaction

**版本**: PolarDB MySQL 8.0.x

业界首个 **FPGA 硬件加速 OLTP 存储引擎 Compaction** (FAST '20 论文)。Compaction 拆为可并行的 Extent 对任务, 流式卸载到 FPGA Compaction Unit。相比 32 线程 CPU, 性能提升 ~25%, CPU 占用降低 ~10%。事务处理流水线化, SIGMOD '19 论文实测相比 RocksDB 提升 31%。支撑 2018 双 11 49.1 万笔/秒销售交易。

#### TerarkDB (Terark → 字节跳动) — 压缩数据上直接搜索

**版本**: v1.4 (RocksDB fork, 原 Terark 公司 2015 年创建, 2019 年被字节跳动收购)

CO-Index (Nested Succinct Trie): 在压缩数据上直接搜索, 无需解压。传统 SSTable 查询需解压 Block → 搜索 → 丢弃; TerarkDB 跳过解压步骤。厂商声称特定场景下有显著提升, 独立测试显示随机读 2.5~4x 提升 (写入场景)。

---

## 六、Compaction 策略演进全景

```
经典策略:
  ├─ Leveled ─── LevelDB, RocksDB 默认, Pebble, HBase Minor
  ├─ Tiered ──── Cassandra STCS, ScyllaDB STCS
  └─ FIFO ────── RocksDB FIFO

混合/统一策略:
  ├─ Universal ── RocksDB → Paimon (数据湖改良)
  ├─ ICS ──────── ScyllaDB (Tiered+Leveled 混合, 5% 空间开销)
  └─ UCS ──────── Cassandra 5.0 (单参数 W 连续调节, 热切换)

架构级优化 (超越 Compaction 策略本身):
  ├─ KV 分离 ──── BadgerDB, TiKV Titan, Pebble ValueSep, RocksDB BlobDB
  ├─ Macro Block ── OceanBase (块级增量, 只重写脏块)
  ├─ 异步/轮转 ── OceanBase 轮转, Paimon 独立 Flink Job
  ├─ FPGA 加速 ── X-Engine
  ├─ 压缩直接搜索 ── TerarkDB CO-Index
  ├─ Deletion Vector ── Paimon / Doris / StarRocks
  └─ MoW 写入时去重 ── Doris / StarRocks / Paimon MOW

趋势:
  1. 策略统一化: 多策略 → UCS/Universal (一个参数调平衡)
  2. KV 分离普及: WiscKey → Titan → Pebble → 越来越多内置
  3. Deletion Vector 标准化: Paimon/Iceberg V3/Delta Lake 共享 Roaring Bitmap
  4. 计算存储解耦: Compaction 独立资源 (Paimon Flink Job / OceanBase 轮转)
  5. 硬件加速: FPGA (X-Engine), 持久内存 (Tair AEP)
```

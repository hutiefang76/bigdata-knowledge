# 主流 LSM-Tree 实现深度对比

> 仅收录使用 LSM-Tree（或 LSM 变体）的组件，非 LSM 系统不在本文范围
>
> 版本信息截至 2026 年 2 月

---

## 目录

- [一、全景对比表](#一全景对比表)
- [二、参考实现](#二参考实现)
  - [LevelDB](#21-leveldb-google)
  - [RocksDB](#22-rocksdb-meta)
- [三、NoSQL 数据库](#三nosql-数据库)
  - [HBase](#31-apache-hbase)
  - [Cassandra](#32-apache-cassandra)
  - [ScyllaDB](#33-scylladb)
- [四、NewSQL 数据库](#四newsql-数据库)
  - [TiDB/TiKV](#41-tidbtikv-pingcap)
  - [OceanBase](#42-oceanbase-蚂蚁集团)
  - [CockroachDB/Pebble](#43-cockroachdbpebble)
- [五、OLAP 引擎](#五olap-引擎)
  - [Doris](#51-apache-doris)
  - [StarRocks](#52-starrocks)
- [六、数据湖 LSM 实现](#六数据湖-lsm-实现)
  - [Paimon](#61-apache-paimon)
  - [Hudi](#62-apache-hudi-类-lsm)
- [七、专用优化引擎](#七专用优化引擎)
  - [BadgerDB](#71-badgerdb-dgraph)
  - [FoundationDB](#72-foundationdb-apple)
  - [PolarDB X-Engine](#73-polardb-x-engine-阿里巴巴)
  - [TerarkDB](#74-terarkdb-字节跳动)
- [八、国内其他 LSM 组件](#八国内其他-lsm-组件)
- [九、设计演进脉络](#九设计演进脉络)
- [十、Compaction 策略族谱](#十compaction-策略族谱)

---

## 一、全景对比表

| 组件 | 最新稳定版 | LSM 来源 | Compaction 策略 | 核心差异化 | 写放大 | 读放大 |
|------|-----------|---------|----------------|-----------|--------|--------|
| **LevelDB** | 1.23 (维护模式) | 原创 | Strict Leveled | 教科书实现，单线程 | 高 | 低 |
| **RocksDB** | ~10.10+ | LevelDB 演进 | Leveled / Universal / FIFO | 多线程, HyperClockCache | 中~高 | 低~中 |
| **HBase** | 2.6.4 | 自研 | Minor + Major | In-Memory Compaction, HDFS 3副本 | 高 | 中 |
| **Cassandra** | 5.0.6 | 自研 | **UCS** (统一策略) | 密度驱动, 无状态, 热切换 | 可调 | 可调 |
| **ScyllaDB** | 2025.4.3 | 自研 (C++) | STCS/LCS/**ICS**/TWCS | Shard-per-core, ICS 混合 | 低~中 | 中 |
| **TiKV** | v8.5.5 LTS | RocksDB 深度定制 | Leveled + Titan GC | SST Guard, Titan KV分离 | 中 | 低 |
| **OceanBase** | V4.3.5 / V4.2.5 | 自研 | Mini→Minor→Major 三级 | 2MB Macro Block, 轮转合并 | 低 | 低 |
| **CockroachDB** | v26.1.0 | Pebble (Go自研) | Leveled (L0~L6) | L0 Sublevel, **Value Separation** | 中→低 | 低 |
| **Doris** | 4.0.3 | 自研 (列式) | Size-based / Time-series | **MoW** Delete Bitmap | 中 | 低 (MoW) |
| **StarRocks** | v4.0.6 | 自研 (列式) | Score-based 调度 | Primary Key + DelVector | 中 | 低 (MoW) |
| **Paimon** | 1.3.x | 自研 (湖格式) | **Universal** (RocksDB 风格) | 唯一 LSM 湖格式, 三模式 | 低 (MOR) | 低 (MOW) |
| **Hudi** | 1.1.1 | 类 LSM (Base+Log) | Inline / Async / Separate | LSM Timeline, LogCompaction | 低 (MOR) | 中 (MOR) |
| **BadgerDB** | v4.x | WiscKey KV 分离 | Leveled (仅Key) + vLog GC | 纯 Go, Value 不参与 Compaction | 极低 | 中 |
| **FoundationDB** | 7.4.5 | 可插拔 (含 RocksDB) | 取决于引擎 | Sharded RocksDB, 物理分片迁移 | 取决于引擎 | 取决于引擎 |
| **X-Engine** | PolarDB 8.0.x | 自研 | Tiered (2-4路) | **FPGA 加速 Compaction**, 流水线事务 | 低 | 低 |
| **TerarkDB** | v1.4 (RocksDB fork) | Terark→字节跳动 | Leveled + ZipTable | **CO-Index** 压缩直接搜索 | 低 | 极低 |

---

## 二、参考实现

### 2.1 LevelDB (Google)

**版本**: 1.23 (2021-02, 维护模式)

LevelDB 是 2011 年 Jeff Dean 和 Sanjay Ghemawat 编写的精简 LSM 实现，~20000 行 C++，定义了业界标准范式。

```
WAL → MemTable (SkipList) → Immutable MemTable → Flush → L0 SSTable
                                                          ↓
                                                  Compaction → L1 → ... → L6
```

**Compaction**: 严格 Leveled。L0 的一个文件与 L1 所有重叠文件合并，限制与 Level N+2 的重叠不超过 10 个 SSTable。

**局限**: 单写线程、单进程锁、无 Column Family、无 Universal/FIFO、无多线程 Compaction。

**意义**: 几乎所有后续 LSM 实现都在这个框架上做变体。

---

### 2.2 RocksDB (Meta)

**版本**: ~10.10+ (月度发布)

RocksDB 是当今 LSM-Tree 的事实标准，被 TiKV、Pegasus、FoundationDB 等直接使用。

#### 三种 Compaction 策略

| 策略 | 写放大 | 读放大 | 空间放大 | 适用场景 |
|------|--------|--------|----------|---------|
| **Leveled** (默认) | 高 (10-30x) | 低 | 低 | 读多写少 |
| **Universal** (Tiered) | 低 (2-7x) | 较高 | 高 (临时 2x) | 写密集 |
| **FIFO** | 最低 | N/A | 最低 | TTL 数据 |

实际上 Leveled 在 L0 是 Tiered (允许重叠)，L1+ 才是 Leveled，属于 **Tiered+Leveled 混合**。

#### 版本演进

| 版本 | 特性 | 影响 |
|------|------|------|
| 5.18 | Titan (PingCAP 插件) | TiKV 用, KV 分离减少大 Value 写放大 |
| 6.18 | Integrated BlobDB | RocksDB 原生 KV 分离, 通过标准 DB API 使用 |
| 6.x | Universal Compaction 成熟 | 写密集场景写放大降低 60%+ |
| 7.x | Tiered Storage 支持 | 冷数据自动迁移到低成本存储 |
| 8.x | Intra-L0 Compaction | L0 内部合并减少读放大 |
| 10.7 | **HyperClockCache** 成默认 | 无锁并发缓存，替代 LRU |
| 10.7 | **并行压缩重构** | CPU 开销降低 65% |

RocksDB 团队正推进 [Unifying Level and Universal Compactions](https://github.com/facebook/rocksdb/wiki/Proposal:-Unifying-Level-and-Universal-Compactions)，与 Cassandra UCS 思路一致。

---

## 三、NoSQL 数据库

### 3.1 Apache HBase

**版本**: 2.5.13 (稳定线) / 2.6.4

```
Client → RegionServer → MemStore (per Column Family) → Flush → HFile (on HDFS) → Compaction
```

**Compaction**:
- **Minor**: StoreFile > threshold 时合并少量相邻 HFile
- **Major**: 定时 (默认 7 天) 或手动合并 Region 内全部 HFile + 清除 tombstone

可选策略: Size-tiered, **Date-tiered**, **Stripe**

**关键优化**:

| 版本 | 特性 | 解决什么放大 |
|-----|------|------------|
| 0.98 | Stripe Compaction | 写放大 (Region 分条带) |
| 1.0 | Date-tiered Compaction | 写放大 (时序数据) |
| 2.0 | In-memory Compaction | 写放大 (MemStore 内先合并) |
| 2.0 | MOB (>100KB) | 写放大 (大 Value 不参与 Compaction) |
| 2.0 | BucketCache (off-heap) | 读放大 (堆外缓存) |

**固有局限**: HDFS 3 副本空间放大、Compaction 与在线共享资源、仅 KV 接口。

---

### 3.2 Apache Cassandra

**版本**: 5.0.6

#### UCS (Unified Compaction Strategy) — 5.0 最大创新

```
核心: 不按 SSTable 大小或固定层级分组，按「数据密度」触发

参数 W (Scaling Parameter):
  W < 0 → Leveled (高写放大, 低读放大)
  W = 0 → 平衡
  W > 0 → Tiered (低写放大, 高读放大)

  不同层可设不同 W
  参数热切换, 无需全量 Compaction — 这是 STCS/LCS 切换做不到的
  无状态: 不依赖元数据做 Compaction 决策
  Shard 级并行
```

**UCS vs 旧策略**:

| 维度 | STCS | LCS | UCS |
|------|------|-----|-----|
| 写放大 | 低 | 高 (10x+) | W 可调 |
| 读放大 | 高 | 低 | W 可调 |
| 切换 | 需全量 Compaction | 需全量 Compaction | **热切换** |
| 状态依赖 | 有 | 有 | **无** |

2025 社区反馈: UCS 替换 LCS 后"压倒性正面"，无一场景 LCS 更优。

**其他 5.0 特性**: Trie Memtable + Trie SSTable, Storage Attached Index (SAI), 向量搜索。

---

### 3.3 ScyllaDB

**版本**: 2025.4.3 (STS) / 2025.1.11 (LTS)

#### Shard-per-Core 架构

每个 CPU 核心拥有独立的内存/存储/网络资源 → 消除线程竞争，Compaction per-shard 执行。

#### Compaction 策略

| 策略 | 写放大 | 空间开销 | 特点 |
|------|--------|---------|------|
| **STCS** (默认) | 低 | 高 (50% 临时) | 写密集 |
| **LCS** | 高 (极端 40x) | 低 | 读密集 |
| **ICS** (企业版, **推荐**) | 低 | **极低 (5%)** | STCS 空间优化 |
| **TWCS** | 低 | 低 | 时序 + TTL |

**ICS (Incremental Compaction Strategy)** — ScyllaDB 独创:
- 大 SSTable 拆为 1GB SSTable Run 段
- Major Compaction 空间开销从 50% → **~5%**
- 跨层 tombstone 清理

---

## 四、NewSQL 数据库

### 4.1 TiDB/TiKV (PingCAP)

**版本**: v8.5.5 LTS (2026-01) / v9.0.0-beta.1

TiKV 运行两个 RocksDB: `raftdb` + `kvdb` (4 Column Family)。

#### RocksDB 深度定制

| 优化 | 原理 | 效果 |
|------|------|------|
| **SST Compaction Guard** | SST 边界对齐 Region 边界 | 避免跨 Region 重写 → 写放大显著降低 |
| **Flow Control** | 替换 Write Stall 为分层限流 | 写入吞吐更平滑 |
| **MVCC GC Filter** | Compaction 底层自动清理旧版本 | 空间放大降低，GC 分布式增量执行 |
| **Periodic Full Compaction** (v7.6+) | 空闲时全量合并 | 清除冗余版本 |

#### Titan KV 分离 (v7.6+ 新集群默认启用)

```
Flush/Compaction 时:
  Value >= 32KB (v7.6+ 默认阈值, 旧版本默认 1KB) → 分离到 Blob File, LSM 只存指针
  Compaction 只移动 Key + 指针

与 WiscKey 的区别: Titan 仍将大 Value 写入 WAL (保证原子性)

性能: Value=1KB → QPS 2x; Value=32KB → QPS 6x
代价: 磁盘空间增大, 大范围 Scan 变慢
```

#### TiDB X (下一代)

Region 所有副本共享逻辑 LSM-Tree，Leader 负责 Compaction，Follower 消费结果 → 根本性消除共享 LSM Compaction 问题。

---

### 4.2 OceanBase (蚂蚁集团)

**版本**: V4.3.5 BP4 (AP LTS) / V4.2.5 BP6 (TP LTS)

OceanBase 自研 LSM-Tree，与 RocksDB 系有**根本性架构差异**。

#### Macro Block — 核心差异

```
RocksDB: Compaction 单位 = 整个 SSTable (数十~数百 MB)
OceanBase: Compaction 单位 = 2MB Macro Block
  → 只重写被修改的 Macro Block, 未修改的直接引用复用
  → 写放大远低于 RocksDB
```

每个 Macro Block 内部由 16KB Micro Block 组成 (读取基本单位)。

#### 三级 Compaction

```
MemTable (双索引: B-Tree MVCC + Hash)
    ↓ dump
Mini SSTable (1~3 个)
    ↓ minor compaction (Mini 积累过多时)
Minor SSTable (最多 1 个)
    ↓ major compaction (每日低峰)
Major SSTable (全量基线)
```

#### 高级策略

| 策略 | 说明 |
|------|------|
| **增量合并** | 默认。只重写脏 Macro Block |
| **渐进合并** | DDL 变更后分 N 次逐步重写 |
| **并行合并** | 多线程并行 Compaction |
| **轮转合并** | **多副本场景独有**: 合并期间查询路由到其他副本 → 彻底隔离对在线查询的影响 |

**混合行列存** (V4.3+): Major Compaction 时基线数据列式化，增量写入仍行式 → 真正 HTAP。

---

### 4.3 CockroachDB / Pebble

**版本**: CockroachDB v26.1.0

Pebble: Go 重写的 LSM，标准 Leveled (L0~L6)。

#### 核心创新

| 特性 | 版本 | 效果 |
|------|------|------|
| **L0 Sublevel** | 早期 | L0 分虚拟子层 → 并发 Compaction 出 L0 → 减少写入堆积 |
| **DeleteSized** | 中期 | tombstone 记录 value 大小 → 更准确的空间放大估算 |
| **Concurrent Manual Compaction** | 中期 | 并行手动 Compaction → 速度 +30% + L0 并行再 +92% |
| **Value Separation** | **v25.4 GA** | 大 Value 存 Blob File → 宽行写密集吞吐 **+50-60%** |

Value Separation 通过 per-SSTable "blob reference depth" 追踪数据局部性，深度超阈值触发 re-compaction。与 TiKV Titan、BadgerDB WiscKey 思路一致。

---

## 五、OLAP 引擎

### 5.1 Apache Doris

**版本**: 4.0.3 / 3.1.3

#### Merge-on-Write (2.1+ 默认)

```
1.x (MoR): 查询时合并多 Rowset → 读放大严重
2.1+ (MoW): 写入时查主键索引 → Delete Bitmap 标记旧行 → 查询直接读
  → 查询性能 10x 提升
  → 代价: 写入多一次查找 (写少读多场景下正确的 trade-off)
```

#### Compaction

| 类型 | 说明 |
|------|------|
| **Cumulative** | 合并小 Rowset |
| **Base** | 合并 Cumulative 结果与 Base Rowset |
| **Segment** (2.1+ 默认) | Rowset 内部 Segment 合并 |
| **Time-Series** | 时序场景, 每文件只参与一次 Compaction |

**其他优化**: 零拷贝 Compaction (BlockView), Idle Schedule (低峰调度), MoW Unique 表禁用 Time-Series Compaction (v2.1.10+)。

---

### 5.2 StarRocks

**版本**: v4.0.6

#### Primary Key 表 = LSM + MoW

```
写入 → 列式 MemTable → Flush → Segment → Rowset
更新 → 内存主键索引定位 → DelVector 标记旧行 → 新数据写入新 Segment → 更新索引
查询 → Segment + DelVector 过滤 → 无需合并多版本
```

#### Score-Based 调度

```
Score < 10   → 无需操作
Score > 100  → 节流写入
Score > 2000 → 拒绝导入
```

**v4.0 云原生**: Smarter Compaction 减少 70-90% 云 API 调用; File Bundling 打包多操作文件。

---

## 六、数据湖 LSM 实现

### 6.1 Apache Paimon

**版本**: 1.3.x (1.3.1, 2025-11)

**唯一在数据湖表格式层面内置 LSM-Tree 的系统。**

```
Table → Partition → Bucket → LSM-Tree (每个 Bucket 独立)
SST 文件格式 = ORC / Parquet (列式)
无 WAL — 依赖 Flink Checkpoint 做崩溃恢复
```

#### 三种表模式

| 模式 | 配置 | 写放大 | 读放大 | 原理 |
|------|------|--------|--------|------|
| **MOR** | 默认 | 低 | 高 | 只做 minor compaction, 读时合并 |
| **COW** | `full-compaction.delta-commits=1` | 高 | 无 | 写入触发全量合并 |
| **MOW** | `deletion-vectors.enabled=true` | 中 | **极低** | LSM 点查 → Deletion Vector → 读时 Bitmap 过滤 |

#### Compaction

**Universal Compaction** (借鉴 RocksDB):
- 触发: sorted run 数 > `num-sorted-run.compaction-trigger`
- 停写: sorted run 数 > `num-sorted-run.stop-trigger`

**Lookup Compaction** (Paimon 独创):
- Radical: 每次强制 L0 合并到更高层
- Gentle: Universal + 可配置最大间隔

**Deletion Vector**: Roaring Bitmap, 多 Bucket 可共享一个 delete file, full compaction 后全部失效。

#### 关键配置

```sql
CREATE TABLE orders (...) WITH (
    'write-buffer-size' = '256mb',
    'compaction.max-sorted-run-num' = '5',
    'deletion-vectors.enabled' = 'true',
    'changelog-producer' = 'lookup'
);

-- Flink 侧
SET 'execution.checkpointing.interval' = '2min';

-- 独立 Compaction Job
CALL sys.compact('db.orders');
```

---

### 6.2 Apache Hudi (类 LSM)

**版本**: 1.1.1 (2025-12) / 1.0 GA (2025-01)

Hudi 的 MOR 在概念上接近 LSM (Base File ≈ 深层 Level, Log File ≈ L0)，但不是严格的 LSM-Tree 实现。

```
File Group:
  Base File (Parquet) + Log File 1 (Avro) + Log File 2 + ...

写入: 追加到 Log File (顺序写)
查询: Base + Log 合并
Compaction: Log → 合并入 Base → 新 File Slice
```

#### 关键创新

| 特性 | 版本 | 说明 |
|------|------|------|
| **LogCompaction** (RFC-48) | 1.0 | Log 间合并不写 Base → 减少写放大 |
| **Record-Level Index** | 1.0 | 元数据表用 HFile (SSTable 格式) → 点查提速 4-10x |
| **LSM Timeline** | 1.x | 时间线元数据 LSM 化 → 支持 1000 万+ 操作 |
| **NBCC** | 1.0 | 多写者并发写 Log, 冲突推迟到 Compaction |
| **RFC-103: Full LSM Layout** | 规划中 | 数据文件全面 LSM 化 |

---

## 七、专用优化引擎

### 7.1 BadgerDB (Dgraph)

**版本**: v4.x

**WiscKey 论文** 的最接近生产级实现。

```
LSM-Tree: 只存 Key + Value Pointer (vptr)
Value Log (vLog): 存实际 Value

Compaction: 只重写 Key + vptr → 写放大接近 1x
代价: 随机读 vLog + vLog GC 开销
```

纯 Go, 无 CGO, SSI 事务。在 Dgraph、Jaeger 等百 TB 级生产使用。

---

### 7.2 FoundationDB (Apple)

**版本**: 7.4.5 (稳定); 8.0 分支已 cut 未正式发布

可插拔存储引擎:

| 引擎 | 数据结构 | 特点 |
|------|---------|------|
| **SSD (SQLite B-tree)** | B-tree | 原始引擎 |
| **Redwood** | CoW B+Tree | 前缀压缩, 更高写吞吐 |
| **RocksDB** | LSM-tree | 标准 RocksDB |
| **Memory** | 内存 | 1GB 上限 |

**8.0 规划 — Sharded RocksDB**:
- 每个 Shard 独立 RocksDB 实例
- 数据迁移 = 文件传输 → 绕过 Compaction
- 物理分片移动, 避免 SST 重写

---

### 7.3 PolarDB X-Engine (阿里巴巴)

**版本**: PolarDB MySQL 8.0.x

**FPGA 加速 Compaction (FAST '20)**:
- Compaction 拆为可并行 Extent 对任务 → 流式卸载到 FPGA
- FPGA CU: Decoder → KV Ring Buffer → Merger → Encoder
- 相比 32 线程 CPU: 性能 +25%, CPU 占用 -10%

**事务处理流水线**: 阶段并行化, SIGMOD '19 论文实测吞吐相比 RocksDB 提升 31%。支撑 2018 双 11 49.1 万笔/秒销售交易 (对应底层 7000 万次数据库事务/秒)。

**热温冷分层**: MemTable (内存) → SSD → HDD/OSS, ZSTD 压缩至 InnoDB 的 1/3~1/10。

---

### 7.4 TerarkDB (Terark → 字节跳动)

**版本**: v1.4 (基于 RocksDB v5.18.3 fork, 原 Terark (北京奇简软件) 2015 年创建, 2019 年被字节跳动收购后开源)

**CO-Index (Compressed Ordered Index)**:
- Nested Succinct Trie: 在压缩数据上直接搜索, 无需解压
- PA-Zip (Point Accessible Zip): 全局压缩 Value, 通过整数 ID 直接提取
- 传统: 解压 Block → 搜索 → 丢弃; TerarkDB: 跳过解压

```
可配置从哪一层开始使用 TerarkZipTable:
  L0~L1: 标准 BlockBasedTable (写入频繁)
  L2+:   TerarkZipTable (数据稳定, CO-Index + PA-Zip)
```

号称 200x+ 性能提升 + 15x+ 存储节省 (厂商早期营销声明; 独立测试显示随机读 2.5~4x 提升)。

---

## 八、国内其他 LSM 组件

### Apache Pegasus (小米)

**版本**: v2.5.0

强一致 KV, 使用官方 RocksDB (v2.5.0 回归官方版), Classic Leveled。支持运行时调整 `num_levels` / `write_buffer_size`。

### Tair (阿里巴巴)

三引擎: MDB (缓存) + RDB (Redis 兼容) + **LDB (LSM 持久化)**。2018 年中国首个生产部署 Intel Optane 持久内存 (AEP)。

### BaikalDB (百度)

**版本**: v2.1.1 (2022-07, 活跃度低)

三层 Share-Nothing 架构, 底层标准 RocksDB Leveled。brpc + braft + RocksDB, ~10 万行 C++。内部千节点 PB 级。

---

## 九、设计演进脉络

```
1996: LSM-Tree 论文 (O'Neil, UMass Boston)
  │
  ├─ 2006: Google Bigtable 论文 (LSM 第一个超大规模实践, 内部系统)
  │    └─ 2007: Apache HBase (Bigtable 开源复刻, 首版随 Hadoop 0.15 发布)
  │
  ├─ 2011: LevelDB (Google, 定义标准范式)
  │    └─ 2012: RocksDB (Facebook, 多线程+Universal → 工业标准)
  │              │
  │              ├─ 2015: TiKV ── SST Guard + Titan + Flow Control
  │              ├─ 2016: BadgerDB ── WiscKey KV 分离
  │              ├─ 2018: TerarkDB ── CO-Index 压缩直接搜索
  │              ├─ 2020: Pebble ── Go 重写 + L0 Sublevel + Value Separation
  │              └─ 推进中: RocksDB 统一 Compaction 提案
  │
  ├─ 2018: X-Engine (阿里) ── FPGA 加速 + 事务流水线
  │
  ├─ 2008: Cassandra (Facebook 开源 → 2009 Apache 孵化)
  │    ├─ 2015: ScyllaDB ── C++ 重写 + Shard-per-Core + ICS
  │    └─ 2024: Cassandra 5.0 UCS ── 统一策略
  │
  ├─ 2010: OceanBase 项目启动 (V0.5 淘宝收藏夹) ── 2014: V1.0 开发 ── 自研 LSM + Macro Block + 轮转合并
  │    └─ 2024: V4.3+ 混合行列存
  │
  ├─ 2008: Doris (百度内部) ── 2017: 开源 → 列式 LSM → 2023: MoW
  ├─ 2020: StarRocks ── Primary Key + DelVector
  │
  └─ 数据湖:
      ├─ 2016: Hudi (Uber 内部) ── 2017: 开源 → MOR (类 LSM) → 2025: LSM Timeline
      └─ 2022: Flink Table Store → 2023: 更名 Paimon ── 唯一湖格式内置 LSM + 三模式
```

---

## 十、Compaction 策略族谱

```
经典策略 (互斥选择):
  ├─ Leveled ──── LevelDB, RocksDB 默认, Pebble, HBase Minor
  ├─ Tiered ───── Cassandra STCS, ScyllaDB STCS
  └─ FIFO ─────── RocksDB FIFO

混合/统一策略:
  ├─ Universal ── RocksDB → Paimon (数据湖改良)
  ├─ ICS ──────── ScyllaDB (Tiered+Leveled 混合, 5% 空间开销)
  └─ UCS ──────── Cassandra 5.0 (单参数 W, 热切换, 无状态)

架构级优化 (超越策略选择):
  ├─ KV 分离 ──── BadgerDB, TiKV Titan, Pebble ValueSep, RocksDB BlobDB
  ├─ Macro Block ── OceanBase (块级增量, 只重写脏块)
  ├─ 异步/轮转 ── OceanBase 轮转, Paimon 独立 Flink Job
  ├─ FPGA 加速 ── X-Engine
  ├─ 压缩直接搜索 ── TerarkDB CO-Index
  ├─ Deletion Vector ── Paimon / Doris / StarRocks
  └─ MoW 写入时去重 ── Doris / StarRocks / Paimon MOW

趋势:
  1. 策略统一化: 多策略 → UCS/Universal
  2. KV 分离普及: WiscKey → Titan → Pebble
  3. Deletion Vector 标准化: Roaring Bitmap 跨系统
  4. 计算存储解耦: Compaction 独立资源池
  5. 硬件加速: FPGA / 持久内存
```

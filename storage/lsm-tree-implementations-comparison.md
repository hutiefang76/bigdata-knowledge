# 主流大数据组件 LSM-Tree 实现深度对比

> 覆盖 18 个组件，从参考实现到 NoSQL / NewSQL / OLAP / 数据湖，含国际与国内大厂方案
>
> 版本信息截至 2026 年 2 月

---

## 目录

- [一、全景对比表](#一全景对比表)
- [二、参考实现](#二参考实现)
  - [LevelDB](#21-leveldb-google--lsm-tree-的教科书原型)
  - [RocksDB](#22-rocksdb-meta--工业级标准实现)
- [三、NoSQL 数据库](#三nosql-数据库)
  - [HBase](#31-apache-hbase--hadoop-生态-lsm)
  - [Cassandra](#32-apache-cassandra--分布式-lsm)
  - [ScyllaDB](#33-scylladb--cassandra-的-c-重写)
- [四、NewSQL 数据库](#四newsql-数据库)
  - [TiDB/TiKV](#41-tidbtikv-pingcap)
  - [OceanBase](#42-oceanbase-蚂蚁集团)
  - [CockroachDB/Pebble](#43-cockroachdbpebble)
- [五、OLAP 引擎](#五olap-引擎)
  - [Doris](#51-apache-doris)
  - [StarRocks](#52-starrocks)
- [六、数据湖表格式](#六数据湖表格式)
  - [Paimon](#61-apache-paimon)
  - [Hudi](#62-apache-hudi)
  - [Iceberg](#63-apache-iceberg)
  - [Delta Lake](#64-delta-lake-databricks)
- [七、专用 / 特殊优化引擎](#七专用--特殊优化引擎)
  - [BadgerDB](#71-badgerdb-dgraph)
  - [FoundationDB](#72-foundationdb-apple)
  - [PolarDB X-Engine](#73-polardb-x-engine-阿里巴巴)
  - [TerarkDB](#74-terarkdb-字节跳动)
- [八、附录：国内其他组件简述](#八附录国内其他组件简述)
- [九、设计演进脉络](#九设计演进脉络)
- [十、Compaction 策略族谱](#十compaction-策略族谱)

---

## 一、全景对比表

| 组件 | 最新稳定版 | LSM 来源 | Compaction 策略 | 核心差异化 | 写放大 | 读放大 |
|------|-----------|---------|----------------|-----------|--------|--------|
| **LevelDB** | 1.23 (维护模式) | 原创 | Strict Leveled | 教科书实现，单线程 | 高 | 低 |
| **RocksDB** | ~10.10+ | LevelDB 演进 | Leveled / Universal / FIFO | 多线程 Compaction, HyperClockCache | 中~高 | 低~中 |
| **HBase** | 2.6.4 | 自研 | Minor + Major | In-Memory Compaction, HDFS 3 副本 | 高 | 中 |
| **Cassandra** | 5.0.6 | 自研 | **UCS** (统一策略) | 密度驱动, 无状态, 热切换 | 可调 | 可调 |
| **ScyllaDB** | 2025.4.3 | 自研 (C++) | STCS / LCS / **ICS** / TWCS | Shard-per-core, ICS 混合策略 | 低~中 | 中 |
| **TiKV** | v8.5.5 LTS | RocksDB 深度定制 | Leveled + Titan GC | SST Guard, Titan KV 分离, Flow Control | 中 | 低 |
| **OceanBase** | V4.3.5 / V4.2.5 | 自研 | Mini→Minor→Major 三级 | 2MB Macro Block 增量合并, 轮转合并 | 低 | 低 |
| **CockroachDB** | v26.1.0 | Pebble (Go 自研) | Leveled (L0~L6) | L0 Sublevel, **Value Separation** | 中→低 | 低 |
| **Doris** | 4.0.3 | 自研 (列式) | Size-based / Time-series | **MoW** Delete Bitmap, 零拷贝 Compaction | 中 | 低 (MoW) |
| **StarRocks** | v4.0.6 | 自研 (列式) | Score-based 调度 | Primary Key + DelVector MoW | 中 | 低 (MoW) |
| **Paimon** | 1.3.x | 自研 (湖格式) | **Universal** (RocksDB 风格) | 唯一 LSM 湖格式, MOR/COW/MOW 三模式 | 低 (MOR) | 低 (MOW) |
| **Hudi** | 1.1.1 | 类 LSM (Base+Log) | Inline / Async / Separate | LSM Timeline, LogCompaction, RLI | 低 (MOR) | 中 (MOR) |
| **Iceberg** | 1.10.1 | 非 LSM (快照) | 手动 rewrite_data_files | V3 Deletion Vector, Row Lineage | 高 (COW) | 低 (V3 DV) |
| **Delta Lake** | 4.1.0 | 非 LSM (COW) | 手动 OPTIMIZE | Deletion Vector, Liquid Clustering | 低 (DV) | 低 (clean) |
| **BadgerDB** | v4.x | WiscKey KV 分离 | Leveled (仅 Key) + vLog GC | 纯 Go, Value 不参与 Compaction | 极低 | 中 |
| **FoundationDB** | 7.4.5 | 可插拔 (RocksDB/Redwood) | 取决于引擎 | Sharded RocksDB, 物理分片迁移 | 取决于引擎 | 取决于引擎 |
| **X-Engine** | PolarDB 8.0.x | 自研 | Tiered (2-4 路) | **FPGA 加速 Compaction**, 事务流水线 | 低 | 低 |
| **TerarkDB** | v1.4 (RocksDB fork) | RocksDB 魔改 | Leveled + ZipTable | **CO-Index** 压缩直接搜索 | 低 | 极低 |

---

## 二、参考实现

### 2.1 LevelDB (Google) — LSM-Tree 的教科书原型

**版本**: 1.23 (2021-02, 维护模式，仅接受关键 bug 修复)

LevelDB 是 LSM-Tree 从论文走向工业界的起点，几乎所有后续实现都是在它基础上演进。

```
架构:
  WAL → MemTable (SkipList) → Immutable MemTable → Flush → L0 SSTable
                                                            ↓
                                                    Compaction → L1 → L2 → ... → L6
```

**Compaction**: 严格 Leveled。L0 的一个文件与 L1 所有重叠文件合并，L1 的一个文件与 L2 重叠文件合并，以此类推。限制合并时与 Level N+2 的重叠不超过 10 个 SSTable。

**局限**:
- 单写线程，单进程锁
- 无 Column Family, 无 Universal/FIFO Compaction
- 无多线程 Compaction
- 无 Bloom Filter 自动管理

**历史意义**: 定义了 MemTable→SSTable→Compaction 的标准范式，后续所有 LSM 组件都在此框架上做变体。

---

### 2.2 RocksDB (Meta) — 工业级标准实现

**版本**: ~10.10+ (月度发布; 10.2.1 Apr 2025 为确认稳定补丁)

RocksDB 是当今 LSM-Tree 的事实标准，被 TiKV、Pegasus、CockroachDB(早期)、FoundationDB 等直接或间接使用。

#### 三种 Compaction 策略

| 策略 | 写放大 | 读放大 | 空间放大 | 适用场景 |
|------|--------|--------|----------|---------|
| **Leveled** (默认) | 高 (10-30x) | 低 | 低 | 读多写少, 存储受限 |
| **Universal** (Tiered) | 低 (2-7x) | 较高 | 高 (临时 2x) | 写密集, 存储充裕 |
| **FIFO** | 最低 | N/A | 最低 | TTL 数据, 缓存 |

实际上 Leveled 策略在 L0 层是 Tiered (允许重叠)，L1+ 才是 Leveled (不重叠)，属于 **Tiered+Leveled 混合**。

#### 版本演进关键特性

| 版本 | 特性 | 影响 |
|------|------|------|
| 5.18 | Titan/BlobDB 集成 | KV 分离减少大 Value 写放大 |
| 6.x | Universal Compaction 成熟 | 写密集场景写放大降低 60%+ |
| 7.x | Tiered Storage 支持 | 冷数据自动迁移到低成本存储 |
| 8.x | Intra-L0 Compaction | L0 内部合并，减少读放大 |
| 10.0 | C++20 要求 | 编译器门槛提高 |
| 10.2 | **HyperClockCache** 成默认 | 替代 LRU, 无锁并发, 高负载下延迟更稳定 |
| 10.7 | **并行压缩重构** | CPU 开销降低 65% |

#### 正在推进的统一方向

RocksDB 团队已提出 [Unifying Level and Universal Compactions](https://github.com/facebook/rocksdb/wiki/Proposal:-Unifying-Level-and-Universal-Compactions)，目标是让 Leveled 和 Universal 在同一个框架内平滑切换——与 Cassandra UCS 的思路一致。

---

## 三、NoSQL 数据库

### 3.1 Apache HBase — Hadoop 生态 LSM

**版本**: 2.5.13 (稳定线, 2025-11) / 2.6.4 (最新 minor)

#### 架构特点

```
Client → RegionServer → MemStore (per Column Family)
                              ↓ Flush
                         HFile (SSTable on HDFS)
                              ↓ Compaction
                         合并后的 HFile
```

每个 Region 的每个 Column Family 有独立的 MemStore 和 HFile 集合。

#### Compaction 策略

| 类型 | 触发条件 | 行为 |
|------|---------|------|
| **Minor** | StoreFile 数量 > `compactionThreshold` (默认 3) | 合并少量相邻 HFile |
| **Major** | 定时 (默认 7 天) 或手动 | 合并 Region 内全部 HFile，清除 tombstone |

可选策略: Size-tiered, **Date-tiered** (时序优化), **Stripe** (Region 内分条带)

#### 关键优化

| 版本 | 特性 | 效果 |
|------|------|------|
| 2.0 | **In-Memory Compaction** | MemStore 内先合并再 Flush → Flush 次数减少 → 写放大降低 |
| 2.0 | **MOB** (Medium Object) | >100KB value 独立存储不参与 Compaction |
| 2.0 | **BucketCache** (off-heap) | 堆外缓存增大容量 → 读放大降低 |

#### 固有问题

- HDFS 3 副本 → 空间放大下限 3x
- Major Compaction 与在线服务共享资源
- 仅 KV 点查和短 Scan，不支持复杂 SQL

---

### 3.2 Apache Cassandra — 分布式 LSM

**版本**: 5.0.6 (2025-10)

#### UCS：统一 Compaction 策略 (5.0 最大创新)

Cassandra 5.0 引入 **Unified Compaction Strategy (UCS)**，用一个策略替代了之前的 STCS/LCS/TWCS 三选一：

```
核心思想: 不按 SSTable 大小或固定层级分组，而是按「数据密度」(每 token 范围单位的数据量) 触发

关键参数 W (Scaling Parameter):
  W < 0  → Leveled 行为 (高写放大, 低读放大)
  W = 0  → 平衡点
  W > 0  → Tiered 行为 (低写放大, 高读放大)

  不同层可设不同 W → 如 L0 用 Tiered, 深层用 Leveled
```

**UCS vs 旧策略**:

| 维度 | STCS | LCS | UCS |
|------|------|-----|-----|
| 写放大 | 低 | 高 (10x+) | W 参数可调 |
| 读放大 | 高 | 低 | W 参数可调 |
| 切换策略 | 需要全量 Compaction | 需要全量 Compaction | **参数热切换，无需重写** |
| 元数据依赖 | 有 | 有 | **无状态** |
| 并行 | 有限 | 有限 | **Shard 级并行** |

2025 年社区反馈: 用 UCS 替换 LCS 后"压倒性正面"，无一场景 LCS 更优。

#### 其他 5.0 特性

- **Trie Memtable + Trie SSTable**: 使用 Trie 数据结构优化内存和存储
- **Storage Attached Index (SAI)**: 革命性索引方式
- 向量搜索支持

---

### 3.3 ScyllaDB — Cassandra 的 C++ 重写

**版本**: 2025.4.3 (STS) / 2025.1.11 (LTS)

#### Shard-per-Core 架构

与 Cassandra 最大的架构区别：每个 CPU 核心拥有独立的内存、存储、网络资源，**消除线程竞争**。Compaction 也是 per-shard 执行。

#### Compaction 策略

| 策略 | 写放大 | 空间开销 | 特点 |
|------|--------|---------|------|
| **STCS** | 低 | 高 (50% 临时) | 默认, 写密集 |
| **LCS** | 高 (40x 极端) | 低 | 读密集 |
| **ICS** (企业版) | 低 | **极低 (5%!)** | STCS 空间优化版, 推荐 |
| **TWCS** | 低 | 低 | 时序 + TTL |

**ICS (Incremental Compaction Strategy)** 是 ScyllaDB 独创：
- 将大 SSTable 拆分为 1GB 的 SSTable Run 段
- Major Compaction 时空间开销从 50% 降到 ~5%
- 支持跨层 tombstone 清理

ScyllaDB 目前正在评估是否需要实现 Cassandra 的 UCS，还是 ICS + 现有策略已经足够。

---

## 四、NewSQL 数据库

### 4.1 TiDB/TiKV (PingCAP)

**版本**: v8.5.5 LTS (2026-01-15) / v9.0.0-beta.1

#### RocksDB 深度定制

TiKV 运行两个 RocksDB 实例：`raftdb` (Raft 日志) + `kvdb` (用户数据, 4 个 Column Family)。在 RocksDB 之上做了大量定制：

| 优化 | 原理 | 效果 |
|------|------|------|
| **SST Compaction Guard** | SST 文件边界对齐 TiKV Region 边界 | 避免一个 SST 跨多个 Region → 写放大显著降低 |
| **Flow Control** | 替换 RocksDB Write Stall 为分层限流 | 写入吞吐更平滑，不会突然卡顿 |
| **MVCC GC via Compaction Filter** | 在 Compaction 底层 Level 自动清理旧版本 | GC 分布式化，增量执行，低影响 |
| **Periodic Full Compaction** (v7.6+) | 空闲时全量合并 | 清除冗余版本 |

#### Titan — KV 分离引擎 (v7.6+ 默认启用)

```
写入路径:
  Key + Value → WAL → MemTable → Flush
                                    ↓
                          Value >= 32KB ?
                          ├── Yes → 分离到 Blob File, LSM 只存 Value 指针
                          └── No  → 正常存入 SSTable

Compaction 时:
  只移动 Key + 指针 (几十字节)
  不移动大 Value (KB~MB)
  → 写放大大幅降低
```

**性能**: Value=1KB 时 QPS 提升 2x, Value=32KB 时提升 6x
**代价**: 磁盘空间增大 (建议预留 2x)，大范围 Scan 性能下降 40%~数倍

#### TiDB X (下一代架构)

Region 所有副本共享一个逻辑 LSM-Tree，Leader 负责 Compaction，Follower 消费结果。从根本上消除共享 LSM 的 Compaction 问题。

---

### 4.2 OceanBase (蚂蚁集团)

**版本**: V4.3.5 BP4 (AP LTS) / V4.2.5 BP6 (TP LTS) / V4.4.1 (共享存储)

OceanBase 的 LSM-Tree 与 RocksDB 系有**根本性架构差异**。

#### Macro Block 设计 — 核心差异

```
RocksDB: Compaction 单位 = 整个 SSTable 文件 (数十~数百 MB)
  → 改一行可能重写整个文件

OceanBase: Compaction 单位 = 2MB Macro Block
  → 只重写被修改的 Macro Block，未修改的直接引用复用
  → 写放大远低于 RocksDB
```

每个 Macro Block 内部由 16KB 的 Micro Block 组成 (读取基本单位)。

#### 三级 Compaction 层次

```
MemTable (内存, 双索引: B-Tree MVCC + Hash)
     ↓ dump
Mini SSTable (1~3 个)
     ↓ minor compaction
Minor SSTable (最多 1 个)
     ↓ major compaction (每日低峰)
Major SSTable (全量基线, 1 个)
```

#### 高级 Compaction 策略

| 策略 | 说明 |
|------|------|
| **增量合并** | 默认。只重写脏 Macro Block，干净的直接引用 |
| **渐进合并** | DDL 变更后分 N 次逐步重写 (如每次 10%) |
| **并行合并** | 多线程并行执行 Compaction 任务 |
| **轮转合并** | 多副本场景下，合并期间查询路由到其他副本，合并完预热缓存后切回 |

#### 核心优势

- **轮转合并**: 利用分布式多副本架构，**彻底隔离 Compaction 对在线查询的影响** — 这是单机 LSM (RocksDB/HBase) 做不到的
- **混合行列存** (V4.3+): Major Compaction 时基线数据转列式，增量写入仍行式 → 真正 HTAP
- 相比 InnoDB 压缩后空间节省 50%~90%

---

### 4.3 CockroachDB / Pebble

**版本**: CockroachDB v26.1.0 (2026-02-18)

Pebble 是 CockroachDB 用 Go 重写的 LSM 引擎 (替代早期的 RocksDB CGO 绑定)。

#### 核心创新

| 特性 | 版本 | 效果 |
|------|------|------|
| **L0 Sublevel** | 早期 | L0 内部分虚拟子层，支持并发 Compaction 出 L0 → 减少写入堆积 |
| **Flush Splitting** | 早期 | Flush 输出在 SST 边界分割 → 更好的 L0 并行 |
| **DeleteSized** | 中期 | tombstone 记录被删 value 大小 → 更准确的空间放大估算 |
| **Concurrent Manual Compaction** | 中期 | 手动 Compaction 并行化 → 速度提升 30% + L0 并行再提升 92% |
| **Value Separation** | v25.4 GA | **最重要的优化** — 大 Value 存入独立 Blob File → 写放大大幅降低 |

#### Value Separation 细节

```
写入:
  Key + Value → MemTable → Flush
                              ↓
                    Value >= 256 bytes?
                    ├── Yes → Value 存入 Blob File, SST 只存 Key + BlobRef
                    └── No  → 正常内联存储

Compaction:
  只移动 Key + BlobRef (几十字节)
  Blob File 通过 "blob reference depth" 追踪数据局部性
  当深度超过阈值 → 触发 re-compaction 重写 Blob

效果: 宽行写密集场景吞吐提升 50-60%
```

这与 TiKV Titan 和 BadgerDB 的 WiscKey 思路一脉相承，但 Pebble 的实现是 Go 原生、无 CGO。

---

## 五、OLAP 引擎

### 5.1 Apache Doris

**版本**: 4.0.3 / 3.1.3

#### Merge-on-Write (2.1+ 默认)

```
Doris 1.x (Merge-on-Read):
  写入 → Rowset 1, 2, ... N (同 key 可能在多个 Rowset)
  查询 → 读所有 Rowset → 运行时合并去重
  问题: 读放大严重

Doris 2.1+ (Merge-on-Write, 默认):
  写入 → 查主键索引 → 旧行标记 Delete Bitmap → 写新行到新 Rowset
  查询 → Bitmap 过滤已删除行 → 直接返回
  效果: 查询性能提升 10 倍
```

#### Compaction 策略

| 类型 | 触发条件 | 行为 |
|------|---------|------|
| **Cumulative** | 小 Rowset 积累 | 合并小 Rowset, 大小差异不宜过大 |
| **Base** | Base/候选比 >= 0.3 | 合并 Cumulative 结果与 Base Rowset |
| **Segment** (2.1+ 默认) | 同 Rowset 内多 Segment | Rowset 内部合并 |
| **Time-Series** | 时序场景 | 相邻时段小文件合并，每文件只参与一次 |

**特殊限制**: MoW Unique 表禁用 Time-Series Compaction (v2.1.10+ / v3.0.6+)

#### 零拷贝 Compaction

Doris 通过 **BlockView** 数据结构实现零拷贝 Compaction 逻辑，在不移动数据的情况下完成合并判断，额外提升约 5% 效率。

---

### 5.2 StarRocks

**版本**: v4.0.6 (2026-02-16)

#### Primary Key 表 = LSM + MoW

```
写入:
  数据 → 列式 MemTable → Flush → Segment 文件 → 多 Segment 组成 Rowset

更新:
  1. 通过内存主键索引定位原始行 (文件号 + 行号)
  2. 在 DelVector 中标记旧行删除
  3. 新数据写入新 Segment
  4. 更新主键索引

查询:
  读 Segment + 应用 DelVector 过滤 → 无需合并多版本
```

#### Score-Based Compaction 调度

StarRocks 用分数机制调度 Compaction:

```
Score < 10   → Compaction 已完成, 无需操作
Score > 100  → 开始节流写入
Score > 2000 → 拒绝导入
```

FE 作为调度器，BE/CN 作为执行器，专用线程池执行。

#### v4.0 云原生优化

- **Smarter Compaction**: 减少 70-90% 云存储 API 调用
- **File Bundling**: 将加载/合并/发布操作的文件打包 → 减少 API 次数
- **元数据缓存 + 文件打包**: 云 API 调用总量降低 90%

---

## 六、数据湖表格式

### 6.1 Apache Paimon

**版本**: 1.3.x (1.3.1, 2025-11)

**唯一在数据湖表格式层面内置 LSM-Tree 的系统。**

#### 架构

```
Table → Partition → Bucket → LSM-Tree (每个 Bucket 独立)

每个 Bucket 内部:
  MemTable → Flush → L0 Sorted Run (ORC/Parquet) → Compaction → L1 → ... → Ln
```

#### 三种表模式

| 模式 | 配置 | 写放大 | 读放大 | 原理 |
|------|------|--------|--------|------|
| **MOR** (默认) | 默认 | 低 | 高 | 只做 minor compaction, 读时合并所有 sorted run |
| **COW** | `full-compaction.delta-commits=1` | 高 | 无 | 每次写入触发全量合并 |
| **MOW** | `deletion-vectors.enabled=true` | 中 | **极低** | 写入时 LSM 点查 → 生成 Deletion Vector (Roaring Bitmap) → 读时 Bitmap 过滤 |

#### Compaction 策略

**Universal Compaction** (借鉴 RocksDB Universal 并改良):

```
触发条件: sorted run 数量 > num-sorted-run.compaction-trigger
停写条件: sorted run 数量 > num-sorted-run.stop-trigger

动态决策: "合并这几个 sorted run 的收益 > 成本?" → 选择性合并
比 Leveled 写放大低 50%+, 比 Tiered 读放大低
```

**Lookup Compaction** (Paimon 独创, 用于 MOW/Lookup Changelog):
- **Radical 模式**: 每次强制将 L0 合并到更高层
- **Gentle 模式**: Universal Compaction + 可配置最大间隔强制 L0 合并

#### Deletion Vector 实现

```
格式: Roaring Bitmap (与 Iceberg V3, Delta Lake 相同编码)
存储: list<bitmap>, 元数据 <filename, offset, size>
粒度: 多个 Bucket 可共享一个 delete file
生命周期: full compaction 后全部失效
```

---

### 6.2 Apache Hudi

**版本**: 1.1.1 (2025-12) / 1.0 GA (2025-01)

#### MOR = 概念上的 LSM

```
File Group:
  Base File (Parquet, 列式, 类似 LSM 深层 Level)
    + Log File 1 (Avro, 行式, 类似 L0)
    + Log File 2
    + ...

写入: 新数据/更新 → 追加到 Log File (顺序写)
查询: 读 Base File + 合并 Log Files (Merge-on-Read)
Compaction: Log Files → 合并入 Base File → 新的 File Slice
```

#### Compaction 部署方式

| 方式 | 说明 |
|------|------|
| **Inline** | 写入完成后同步执行 → 最简单, 但阻塞写入 |
| **Async** (同进程) | 写入线程 + 独立 Compaction 线程 → 准实时 |
| **Separate Process** | 独立计算资源 → 写入零影响 |

#### 关键创新

| 特性 | 版本 | 说明 |
|------|------|------|
| **Record-Level Index (RLI)** | 1.0 | 元数据表本身用 HFile (SSTable 格式) 存储, 支持快速点查 → 索引查找提速 4-10x |
| **LogCompaction** (RFC-48) | 1.0 | Log Block 之间先合并 (不写入 Base), 减少 Base File 重写 → 写放大降低 |
| **LSM Timeline** | 1.x | 时间线元数据重构为 LSM-Tree 布局 → 支持 1000 万+ 时间线操作 |
| **NBCC** | 1.0 | 多写者并发写 Log File, 冲突推迟到 Compaction 解决 |
| **RFC-103: Full LSM Layout** | 规划中 | 将数据文件组织全面 LSM 化 → 排序写入 + N-way 合并 |

---

### 6.3 Apache Iceberg

**版本**: 1.10.1 (2025-12)

#### V2 → V3 删除机制演进

| 机制 | 版本 | 写放大 | 读放大 | 状态 |
|------|------|--------|--------|------|
| **Position Delete** | V2 | 低 | 中 (需读 delete file) | V3 中已废弃 |
| **Equality Delete** | V2 | 低 | **极高** (累积恶化) | 不推荐 |
| **Deletion Vector** | **V3** | 低 | **低** (Bitmap 过滤) | 当前推荐 |

V3 Deletion Vector:
- 使用 **Puffin 文件** 存储 Roaring Bitmap
- 每个数据文件每个快照最多一个 DV
- 写入时必须合并新旧 DV
- 与 Delta Lake 共享编码格式 → 跨格式互通

#### Compaction

```sql
-- 手动触发 (Spark/Flink)
CALL catalog.system.rewrite_data_files(
    table => 'db.orders',
    strategy => 'sort',
    sort_order => 'order_id'
);

-- 清理过期快照
CALL catalog.system.expire_snapshots(
    table => 'db.orders',
    older_than => TIMESTAMP '2026-02-21 00:00:00'
);
```

**Iceberg 不自动触发 Compaction**，完全依赖外部引擎调度。

---

### 6.4 Delta Lake (Databricks)

**版本**: 4.1.0 (2026-02-20)

#### 定位: 非 LSM, COW + Deletion Vector

Delta Lake 不是 LSM-Tree，是 COW 模型 + DV 优化层：

```
不带 DV:
  UPDATE → 重写整个 Parquet 文件 (COW) → 写放大极高

带 DV:
  UPDATE → 旧行标记 Roaring Bitmap → 新行写新文件 → 写放大极低
  但 DV 累积后读性能下降 → 需定期 OPTIMIZE 物理清理
```

#### Liquid Clustering (替代分区 + ZORDER)

```sql
CREATE TABLE orders CLUSTER BY (region, create_date);
-- 使用 Hilbert 曲线多维聚类
-- 可以在线修改聚类列, 不重写历史数据
-- 4.0+ 支持 CLUSTER BY AUTO (自动选择聚类列)
```

#### 4.0/4.1 关键特性

- **Coordinated Commits**: 跨环境安全写入
- **Variant 数据类型**: 半结构化 Schema-on-Read
- **VACUUM LITE**: 基于事务日志而非目录扫描 → 更快清理
- **Delta Kernel Rust 0.1**: Rust 原生读写

---

## 七、专用 / 特殊优化引擎

### 7.1 BadgerDB (Dgraph)

**版本**: v4.x (2026-02)

BadgerDB 是 **WiscKey 论文** 的最接近生产级实现，核心思想：KV 彻底分离。

```
LSM-Tree 只存: Key + Value Pointer (vptr)
Value Log (vLog) 存: 实际 Value

Compaction 时:
  只重写 Key + vptr (几十字节)
  大 Value 完全不动
  → 写放大极低 (接近 1x)

代价:
  随机读 vLog 取 Value → 读放大略高
  vLog GC 需要额外维护
```

纯 Go 实现，无 CGO 依赖，支持 SSI 事务。在 Dgraph、Jaeger 等系统中百 TB 级生产使用。

---

### 7.2 FoundationDB (Apple)

**版本**: 7.4.5 (2025-09, 稳定); 8.0 分支已 cut 但未正式发布

FoundationDB 的独特之处：**可插拔存储引擎**。

| 引擎 | 数据结构 | 特点 |
|------|---------|------|
| **SSD (SQLite B-tree)** | B-tree | 原始引擎, 为 SSD 优化 |
| **Redwood** | Copy-on-Write B+Tree | 替代 SQLite, 前缀压缩, 更高写吞吐 |
| **RocksDB** | LSM-tree | 标准 RocksDB, 可调压缩和 Compaction |
| **Memory** | 内存 + append-only log | 1GB 上限, 测试用 |

#### 8.0 规划 (Sharded RocksDB)

```
当前: 一个 Storage Server 一个 RocksDB 实例
       → 数据迁移 = 读取 + 网络传输 + 重写 + Compaction

8.0: 每个 Shard 一个 RocksDB 实例
     → 数据迁移 = 文件传输 (绕过 Compaction!)
     → 物理分片移动, 避免 SST 重写
```

---

### 7.3 PolarDB X-Engine (阿里巴巴)

**版本**: PolarDB for MySQL 8.0.1 / 8.0.2

#### 三大核心创新

**1. FPGA 加速 Compaction (FAST '20)**

```
传统: CPU 线程执行 Compaction
X-Engine: Compaction 拆分为可并行的 Extent 对任务 → 流式卸载到 FPGA

FPGA Compaction Unit (CU) 内部:
  Decoder → KV Ring Buffer → KV Transfer → Key Buffer → Merger → Encoder

效果: 相比 32 线程 CPU compaction, 性能提升 ~25%, CPU 占用降低 ~10%
```

这是业界首次将 FPGA 硬件加速应用于 OLTP 存储引擎的 Compaction。

**2. 事务处理流水线**

将事务处理阶段并行化（类似 CPU 流水线），吞吐量相比 RocksDB 提升 10x+。支撑 2018 双 11 开场 49.1 万笔/秒交易。

**3. 热温冷分层**

```
Hot:  MemTable (内存, 无锁索引 + Append-Only)
Warm: L0~L1 (SSD)
Cold: L2+ (SSD/HDD/OSS, ZSTD 压缩)

相比 InnoDB 存储空间压缩至 1/3 ~ 1/10
```

---

### 7.4 TerarkDB (字节跳动)

**版本**: v1.4 (基于 RocksDB v5.18.3 fork, 内部持续使用)

#### CO-Index: 压缩数据上直接搜索

TerarkDB 的核心创新是 **Compressed Ordered Index**:

```
传统 SSTable:
  Data Block (压缩存储) → 查询时解压 → 在解压数据中搜索 → 丢弃
  Block Cache 存的是解压后的数据 → 缓存效率低

TerarkDB:
  CO-Index (Nested Succinct Trie) → 在压缩数据上直接搜索, 无需解压
  PA-Zip (Point Accessible Zip) → 全局压缩 Value, 通过整数 ID 直接提取单个 Value

效果:
  "传统系统只能索引 1%, 我们能索引 100%"
  → 接近信息论下界的空间占用 + 直接搜索压缩数据
```

#### 架构

```
可配置从哪一层开始使用 TerarkZipTable:
  L0~L1: 标准 BlockBasedTable (写入频繁, 不值得重压缩)
  L2+:   TerarkZipTable (数据稳定, CO-Index + PA-Zip 全局压缩)
```

性能声称: 相比 LevelDB/RocksDB, 200x+ 性能提升 + 15x+ 存储节省 (特定场景)。

---

## 八、附录：国内其他组件简述

### Apache Pegasus (小米)

**版本**: v2.5.0

强一致性分布式 KV 存储，使用**官方 RocksDB** (v2.5.0 起回归官方版, 移除早期定制)。Classic Leveled Compaction。支持运行时调整 `num_levels` 和 `write_buffer_size`。

### Tair (阿里巴巴)

云服务持续演进。三引擎架构: MDB (缓存) + RDB (Redis 兼容) + **LDB (LSM-Tree 持久化)**。2018 年中国首个生产部署 Intel Optane 持久内存 (AEP)。

### BaikalDB (百度)

**版本**: v2.1.1 (2022-07, 开源活跃度低)。三层 Share-Nothing 架构, 底层用标准 RocksDB Leveled Compaction。基于 brpc + braft + RocksDB，约 10 万行 C++ 代码。内部千节点 PB 级部署。

---

## 九、设计演进脉络

```
1996: LSM-Tree 论文 (Patrick O'Neil)
  │
  ├─ 2011: LevelDB (Google) ─── 定义了 MemTable→SSTable→Compaction 范式
  │         │
  │         ├─ 2012: RocksDB (Facebook) ─── 多线程 + Column Family + Universal Compaction
  │         │         │
  │         │         ├─ 2015: TiKV ─── SST Guard + Titan + Flow Control
  │         │         ├─ 2016: BadgerDB ─── WiscKey KV 分离 (走极端: Value 完全不参与 Compaction)
  │         │         ├─ 2018: TerarkDB ─── CO-Index (走另一个极端: 压缩上直接搜索)
  │         │         ├─ 2020: Pebble (CockroachDB) ─── Go 重写 + L0 Sublevel + Value Separation
  │         │         └─ 2024: RocksDB 统一 Compaction 提案
  │         │
  │         └─ 2018: X-Engine (阿里巴巴) ─── FPGA 硬件加速 + 事务流水线
  │
  ├─ 2008: HBase ─── Region 级 LSM + HDFS
  │         │
  │         ├─ 2009: Cassandra ─── 分布式 LSM + 多 Compaction 策略
  │         │         │
  │         │         ├─ 2015: ScyllaDB ─── C++ 重写 + Shard-per-Core + ICS
  │         │         └─ 2024: Cassandra 5.0 UCS ─── 统一策略, 用一个参数 W 控制读写放大平衡
  │         │
  │         └─ 2014: Pegasus (小米) ─── RocksDB 上加 Raft 一致性
  │
  ├─ 2014: OceanBase V1 ─── 自研 LSM + 2MB Macro Block + 三级 Compaction + 轮转合并
  │         └─ 2024: V4.3+ 混合行列存 ─── Compaction 时基线数据列式化
  │
  ├─ 2017: Doris (百度 Palo 开源) ─── 列式 LSM + MoR
  │         └─ 2023: Doris 2.1+ MoW ─── Delete Bitmap 写入时去重
  │
  ├─ 2020: StarRocks ─── 在 Doris 基础上 Primary Key + DelVector
  │
  └─ 数据湖:
      ├─ 2017: Hudi ─── MOR (Base + Log, 类 LSM 概念)
      │         └─ 2025: 1.0 GA LSM Timeline + LogCompaction
      ├─ 2018: Iceberg ─── 快照模型 (非 LSM)
      │         └─ 2025: V3 Deletion Vector
      ├─ 2019: Delta Lake ─── COW (非 LSM)
      │         └─ 2024: Deletion Vector + Liquid Clustering
      └─ 2023: Paimon ─── 唯一在湖格式层面内置 LSM + 三模式 (MOR/COW/MOW)
```

---

## 十、Compaction 策略族谱

```
经典策略 (互斥选择):
  ├─ Leveled ──────── LevelDB (1种), RocksDB, Pebble, HBase Minor
  ├─ Size-Tiered ──── Cassandra STCS, ScyllaDB STCS, HBase
  ├─ FIFO ─────────── RocksDB FIFO
  └─ Time-Window ──── Cassandra TWCS, ScyllaDB TWCS, Doris Time-Series

混合/改良策略 (在经典策略间找平衡):
  ├─ Universal ─────── RocksDB Universal → Paimon Universal (数据湖改良)
  ├─ ICS ──────────── ScyllaDB 独创 (Tiered + Leveled 混合, 5% 空间开销)
  ├─ UCS ──────────── Cassandra 5.0 独创 (单参数 W 连续调节, 无状态, 热切换)
  └─ 统一提案 ─────── RocksDB 正在推进 Leveled↔Universal 统一框架

架构级策略 (超越 Compaction 本身):
  ├─ KV 分离 ────────── BadgerDB (WiscKey), TiKV Titan, Pebble Value Separation, RocksDB BlobDB
  │                      核心: 大 Value 不参与 Compaction → 写放大从根本降低
  ├─ Macro Block ────── OceanBase (2MB 块级增量合并 → 只重写脏块)
  ├─ 异步/轮转 ─────── OceanBase 轮转合并, Paimon 独立 Flink Job
  ├─ FPGA 加速 ─────── X-Engine (硬件卸载 Compaction 执行)
  ├─ 压缩直接搜索 ──── TerarkDB CO-Index (消除解压开销)
  ├─ Deletion Vector ── Paimon / Iceberg V3 / Delta Lake (Bitmap 标记替代文件重写)
  └─ MoW (写入时去重) ── Doris / StarRocks / Paimon MOW (写入多一步查找, 读时零合并)

趋势:
  1. 策略统一化: STCS/LCS/TWCS → UCS (Cassandra) / Universal (RocksDB/Paimon)
  2. KV 分离普及: WiscKey → Titan → Pebble → 越来越多系统内置
  3. Deletion Vector 标准化: Paimon/Iceberg V3/Delta Lake 共享 Roaring Bitmap 编码
  4. 计算存储解耦: Compaction 独立资源池 (Paimon + Flink Job / OceanBase 轮转)
  5. 硬件加速: FPGA (X-Engine), 持久内存 (Tair AEP), GPU (探索中)
```

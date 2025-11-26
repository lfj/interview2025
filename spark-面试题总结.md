# Spark 面试题总结（Markdown 版）

> 覆盖核心架构、数据抽象、执行优化、SQL 与流处理、Shuffle 与分区、存储与序列化、资源与部署、性能调优与故障排查、湖仓生态与实践题。强调高频追问与作答要点。

## 目录
- 基础与架构
- 数据抽象（RDD/DataFrame/Dataset）
- 执行引擎（Catalyst/Tungsten/AQE）
- 任务、Stage 与调度
- Shuffle、Join 与分区策略
- 存储、序列化与缓存
- Spark SQL 与优化
- Structured Streaming
- 文件格式与湖仓（Parquet/ORC/Delta/Iceberg/Hudi）
- 部署与资源管理（Standalone/YARN/K8s）
- 性能调优与实践
- 故障排查与可观测性
- 高频面试题（含要点）
- 实战动手题
- 命令与配置速查

---

## 基础与架构
- 核心组件：Driver、Executor、Cluster Manager（Standalone/YARN/K8s）、SparkContext/SparkSession
- 作业执行：逻辑计划 → 物理计划 → 任务划分（Stage/Task）→ 调度与执行
- 容错机制：RDD 血统（Lineage）与窄依赖/宽依赖；失败重算与 Checkpoint

## 数据抽象（RDD/DataFrame/Dataset）
- RDD：不可变分布式集合，函数式 API（map/filter/reduce），强控制、弱优化
- DataFrame：带 Schema 的分布式表，基于 Catalyst 优化；动态类型（Row）
- Dataset（Scala）：强类型封装 + Catalyst；在 JVM 端保留类型安全
- 选择：多数场景优先 DataFrame/Dataset，RDD 适合自定义复杂算子或细粒度控制

## 执行引擎（Catalyst/Tungsten/AQE）
- Catalyst：查询优化器，规则优化（谓词下推、列裁剪、常量折叠）、代价优化（CBO）
- Tungsten：内存与执行优化，堆外内存、二进制内存布局、代码生成（Whole‑Stage Codegen）
- AQE（Adaptive Query Execution）：自适应执行（动态合并 Shuffle 分区、自动选择 Join 策略、处理数据倾斜）
  - 关键配置：`spark.sql.adaptive.enabled=true`

## 任务、Stage 与调度
- Stage 切分：宽依赖（Shuffle）作为边界；窄依赖可在同一 Stage 合并
- Task 粒度：分区级任务；任务数由父 RDD 分区、Shuffle 分区等决定
- 调度：FIFO 与 FAIR，资源隔离与池化；本地性（PROCESS/EXECUTOR/NODE/RACK）

## Shuffle、Join 与分区策略
- Shuffle 类型：Hash、Sort（Spark 2.x 起默认 SortShuffle）；引发磁盘与网络 IO、内存压力
- Join 策略：Broadcast Hash Join、Sort Merge Join、Shuffle Hash Join；依据大小与排序性选择
  - 自动广播阈值：`spark.sql.autoBroadcastJoinThreshold`（常设 10–30MB，视资源而定）
- 倾斜处理：AQE Skew Join、加盐（Salting）、自定义分区器、拆分热点键、预聚合
- 分区控制：`repartition/coalesce`、`partitionBy`（写入）、合理设置 `spark.sql.shuffle.partitions`（默认 200，按数据与资源调优）

## 存储、序列化与缓存
- 文件格式：Parquet/ORC（列式、压缩、谓词下推），CSV/JSON（文本，性能弱）
- 序列化：Kryo vs Java 序列化；Kryo 更快更小，需注册类（`spark.serializer=org.apache.spark.serializer.KryoSerializer`）
- 缓存与持久化：`cache/persist(StorageLevel)`；缓存对象化 vs 原始列式；谨慎缓存超大数据
- Checkpoint：打破血统以降低重算成本；区分 RDD Checkpoint 与流式 Checkpoint

## Spark SQL 与优化
- Schema 与类型：明确字段类型避免 Catalyst 反复推断；UDF 慎用（难以优化与矢量化）
- 谓词下推与列裁剪：确保数据源（Parquet/ORC）支持；过滤尽量前置
- 代码生成与并行度：启用 Whole‑Stage Codegen；合理设置 Shuffle 分区与并发任务
- 统计信息与 CBO：`ANALYZE TABLE/SQL` 收集统计，提高 Join/物理计划选择准确度

## Structured Streaming
- 微批（micro‑batch）与连续处理（experimental）；统一批流 API
- 状态与窗口：事件时间、Watermark、防止状态膨胀；聚合与会话窗口
- Exactly‑once：端到端取决于 Source/Sink 支持；Kafka Source + 事务性 Sink（或幂等写入）
- 触发器：`trigger(ProcessingTime/Once/Continuous)`；输出模式（Append/Update/Complete）
- 容错与恢复：Checkpoint 目录、WAL；回放与状态恢复
- 反压与限速：`maxOffsetsPerTrigger`、源/下游背压；避免下游慢导致状态积压

## 文件格式与湖仓（Parquet/ORC/Delta/Iceberg/Hudi）
- Parquet/ORC：列式、压缩、字典编码、谓词下推，最常用 OLAP 格式
- Delta Lake：ACID、索引与事务日志、流批一体；`MERGE INTO`、时间旅行
- Apache Iceberg：表格式、快照与分区演进；隐藏分区；多引擎
- Apache Hudi：增量拉取、Upsert、变更捕获（CDC）友好
- 场景选型：ACID 与变更多 → Delta/Iceberg/Hudi；只读分析 → Parquet/ORC

## 部署与资源管理（Standalone/YARN/K8s）
- Cluster Manager：Standalone（简单）、YARN（Hadoop 生态）、K8s（云原生）
- 资源配置：Driver/Executor 数量、核数（`--executor-cores`）、内存（`--executor-memory`）、并行度（总核数 × 每核任务数）
- 动态资源：`spark.dynamicAllocation.enabled` 与 Shuffle Service；K8s 下使用 Operator/Helm 管理
- 提交与模式：`client` vs `cluster`；Jar/依赖打包与分发

## 性能调优与实践
- 数据裁剪与过滤前置：减少读取与 Shuffle 数据量
- 并行度：根据数据量与资源设置 `spark.sql.shuffle.partitions`、`spark.default.parallelism`
- 广播与缓存：小维表广播；热维表缓存（注意一致性）；谨慎 Cache 大对象
- 序列化与内存：Kryo、压缩、堆外内存；GC 监控与对象复用（`mapPartitions`）
- 倾斜治理：AQE、Salting、Hot Key 拆分、预聚合与分层计算
- I/O：选择列式格式、压缩编码；合理文件大小（避免小文件/过大文件）
- UDF/UDAF：能用内置函数尽量用；UDF 切换到 pandas UDF（Arrow）在 PySpark 提升性能

## 故障排查与可观测性
- Spark UI：Stage/Task/Executor 指标、Shuffle 读写、时间线；查看慢任务与倾斜分区
- 事件日志：`spark.eventLog.enabled=true`；历史服务器复盘作业
- 常见问题：
  - OOM（Executor/Driver）：数据过大/缓存过多/广播表过大；调整内存与分区，清理 Cache
  - 数据倾斜：少数键流量集中；开启 AQE、Salting、拆分热点
  - 小文件/元数据压力：合并输出、`coalesce/repartition`、使用湖仓表管理
  - Shuffle 失败：磁盘不足、过多并发；调整并发与资源、优化分区数

## 高频面试题（含要点）
- RDD 与 DataFrame/Dataset 的差异与选择？
  - DF/DS 受 Catalyst 优化且更高层；RDD 强控制但优化弱；多数场景用 DF/DS
- Catalyst/Tungsten 如何优化执行？
  - 规则与代价优化、谓词下推/列裁剪、Whole‑Stage Codegen、堆外内存与二进制布局
- 什么是窄依赖/宽依赖？如何影响 Stage 划分与性能？
  - 窄依赖可流水线，同一 Stage；宽依赖需 Shuffle，产生 Stage 边界与 IO
- Join 策略如何选择？广播阈值与 AQE 的作用？
  - 小表广播优先；SMJ 适合已排序/可排序大表；AQE 动态选择与合并分区
- 倾斜如何定位与治理？
  - UI 查看分区耗时/数据量；Skew Join、Salting、分层聚合、重分区
- Structured Streaming 的 Exactly‑once 如何保证？
  - 依赖 Source/Sink；Kafka 事务性写或幂等 Sink；Checkpoint 管理状态
- Kryo vs Java 序列化何时选？
  - 性能优先选 Kryo；需注册类；跨语言或复杂对象慎重评估
- 文件格式为何偏向 Parquet/ORC？
  - 列式、压缩、谓词下推；读取更少数据、更快
- 动态资源与并行度如何设置？
  - 根据集群资源与作业规模设定分区与核数；启用动态分配需 Shuffle Service 支持

## 实战动手题
- 使用 AQE 自动合并小分区与选择 Join 策略，比较开启/关闭对大表 Join 的影响
- 写一个处理倾斜的聚合：对热点键进行加盐后聚合，再还原
- Structured Streaming 从 Kafka 读取，基于事件时间与 Watermark 做会话窗口聚合，写入事务性 Sink
- 将 CSV/JSON 批量转 Parquet，开启列裁剪与谓词下推，评估读取性能提升
- 调优作业：收集统计信息，调整 `spark.sql.shuffle.partitions` 与广播阈值，观察 UI 指标变化

## 命令与配置速查
- 提交：
```bash
spark-submit \
  --class com.example.App \
  --master yarn \
  --deploy-mode cluster \
  --num-executors 8 \
  --executor-cores 4 \
  --executor-memory 8G \
  --conf spark.sql.adaptive.enabled=true \
  --conf spark.sql.shuffle.partitions=400 \
  app.jar
```
- 配置：
  - `spark.sql.adaptive.enabled=true` 开启 AQE
  - `spark.sql.autoBroadcastJoinThreshold=30MB` 广播阈值
  - `spark.serializer=org.apache.spark.serializer.KryoSerializer` 启用 Kryo
  - `spark.sql.shuffle.partitions=200` 基准值，按数据/资源调优
  - `spark.default.parallelism` 并行度（RDD）

---

如需生成“逐题参考答案”或基于你的数据栈（YARN/K8s、Delta/Iceberg、Kafka/Streaming）的专项题库，我可以继续扩展并加入示例代码与 UI 指标解析说明。

---

## 逐题参考答案（精简版）
- RDD/DF/DS 选择
  - 绝大多数 SQL/ETL：DF/DS（Catalyst 优化更强）
  - 需要自定义复杂算子/细粒度控制：RDD
- Catalyst/Tungsten 优化
  - 规则优化（谓词下推/列裁剪）+ 代价优化（CBO/统计），代码生成 + 堆外内存 + 二进制布局提升执行效率
- 窄/宽依赖与 Stage
  - 窄依赖流水线同一 Stage；宽依赖需 Shuffle，产生 Stage 边界与磁盘/网络 IO
- Join 策略与广播
  - 小表广播优先（BHJ）；SMJ 适合可排序大表；开启 AQE 动态选择与合并分区
- 倾斜治理
  - AQE SkewJoin、加盐拆热点、预聚合分层、`repartition` 控制分区数
- Streaming Exactly‑once
  - Kafka Source + 事务性或幂等 Sink + Checkpoint；端到端取决于 Source/Sink 实现
- Kryo vs Java 序列化
  - 性能优先 Kryo（需注册类）；Java 序列化备用但体积大、性能低
- 文件格式选型
  - 列式（Parquet/ORC）具备压缩/谓词下推；Delta/Iceberg 适合 ACID 与变更场景
- 动态资源与并行度
  - `spark.sql.shuffle.partitions` 与 `spark.default.parallelism` 根据总核数/数据量调优；动态分配需 Shuffle Service

## 示例代码
### Broadcast Join（Scala）
```scala
val dfLarge = spark.table("fact")
val dfSmall = spark.table("dim").hint("broadcast")
val joined = dfLarge.join(dfSmall, Seq("key"), "left")
joined.write.mode("overwrite").parquet("/tmp/out")
```

### 处理倾斜（加盐示例，Scala）
```scala
import org.apache.spark.sql.functions._
val saltN = 8
val salted = fact.withColumn("salt", expr(s"cast(rand()*$saltN as int)"))
val factSalt = salted.select(col("key"), col("salt"), col("value"))
val dimExpanded = dim.select(col("key"))
  .crossJoin(spark.range(saltN).toDF("salt"))
val joined = factSalt.join(dimExpanded, Seq("key","salt"))
val agg = joined.groupBy("key").agg(sum("value").as("sum_v"))
```

### Structured Streaming（事件时间 + Watermark）
```scala
val input = spark.readStream.format("kafka")
  .option("kafka.bootstrap.servers", "broker:9092")
  .option("subscribe", "topic").load()
val parsed = input.selectExpr("CAST(value AS STRING)").as[String]
  .selectExpr("split(value, ',')[0] as key", "timestamp(value) as ts", "cast(split(value, ',')[1] as double) as v")
val agg = parsed
  .withWatermark("ts", "10 minutes")
  .groupBy(window(col("ts"), "5 minutes"), col("key"))
  .agg(sum("v").as("sum_v"))
agg.writeStream
  .format("delta")
  .option("checkpointLocation", "/ckpt/stream")
  .outputMode("update")
  .start("/delta/out")
```

## Spark UI 指标解读
- Stage/Task：查看长尾分区与失败重试；任务耗时分布识别倾斜
- Shuffle Read/Write：字节与时间；过大表示分区设置或上游过滤不足
- Executors：GC 时间、内存与存储占用，判断缓存是否溢出或对象过多
- SQL 面板：物理计划（Whole‑Stage Codegen）、广播标记、AQE 合并分区/Skew 处理

## 专项题库（YARN/K8s/湖仓/Streaming）
- YARN
  - Client vs Cluster 模式差异与适用场景
  - 内存与核数配置策略，`spark.executor.instances` 与并行度的关系
- K8s
  - Driver/Executor Pod 生命周期、镜像与依赖分发；Operator/Helm 部署
  - 动态分配在 K8s 的限制与替代方案（K8s 级 HPA）
- 湖仓（Delta/Iceberg/Hudi）
  - CDC/Upsert 实现路径；`MERGE INTO` 的性能与分区演进
  - 小文件治理：`OPTIMIZE`/`RewriteDataFiles` 与快照管理
- Streaming
  - 反压与速率控制、Watermark 与迟到数据；Exactly‑once 落地与事务性 Sink

## 调优检查清单（上线前）
- 开启 AQE；设置合理 `spark.sql.shuffle.partitions`
- 小维表广播；收集统计信息启用 CBO
- 文件格式与大小：Parquet/ORC，单文件 128–512MB；避免小文件风暴
- 序列化：Kryo；必要时注册类；使用列式数据减少对象化
- 监控与日志：开启事件日志，设置历史服务器；UI 指标告警（长尾与倾斜）

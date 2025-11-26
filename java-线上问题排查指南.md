# Java 线上问题排查指南（含 Arthas）

> 面向线上快速定位与缓解，覆盖性能异常、稳定性问题、资源耗尽、依赖失效与容器环境。重点提供 Arthas 的实战用法与命令速查。

## 快速响应
- 影响评估：错误率、延迟分位数、QPS、影响用户比例、是否扩散
- 防护与降级：开启限流/熔断/降级；暂停灰度或快速回滚
- 证据采集：日志片段、线程堆栈、GC 状态、系统资源快照、依赖健康；必要时临时提高日志级别

## 可观测性基础
- 日志：结构化、`traceId` 贯穿；明确错误分类（客户端/服务端/依赖）
- 指标：Micrometer/Prometheus（QPS、错误率、p95/p99、线程池、连接池、GC、堆/非堆）
- 链路：SkyWalking/Zipkin/Jaeger；关注跨服务瓶颈与重试放大
- 变更：发布/配置改动/依赖故障/流量峰值/节点调度

## CPU 异常（高占用/飙升）
- 定位步骤：
  - `top -H -p <pid>` 查看热点线程 `tid` → `printf '%x\n' <tid>` 得到十六进制 `nid`
  - `jstack <pid> | grep -A 50 'nid=0x<hex>'` 查看对应线程栈；或 `jcmd <pid> Thread.print`
  - 采样分析：`async-profiler` 火焰图（`./profiler.sh -d 30 -e cpu <pid>`）
- 常见根因：死循环/错误重试风暴、阻塞计算、频繁 GC
- 修复建议：限速与熔断、修正循环/重试策略、转向内存问题排查

## 内存问题（OOM/泄漏/频繁 GC）
- 快照与信息：
  - 堆转储：`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/dumps`
  - 即时查看：`jcmd <pid> GC.heap_info`、`jstat -gcutil <pid> 1s 10`
  - 分析 dump：MAT（Leak Suspects、Dominators），定位大对象与持有者链
- 分类与修复：
  - Heap OOM：集合无界、缓存不淘汰、对象爆增；加边界、LRU、批处理/分页
  - Metaspace OOM：类加载泄漏（热加载/字节码生成），检查 ClassLoader 生命周期
  - 线程/直接内存：未关闭资源、`ByteBuffer` 直接内存泄漏；确保 `close()`、降低上限
- GC 调优：
  - GC 日志：`-Xlog:gc*:file=gc.log:time,uptime,level,tags`
  - G1 建议：`-XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+ParallelRefProcEnabled`
  - 观察 `Young`/`Mixed` 比例、停顿时长、晋升失败

## 延迟与超时（接口慢/超时）
- 快速查点：
  - 线程池饱和：业务/IO 线程池队列长度、活动线程；临时扩容并压测验证
  - 外部依赖阻塞：DB/缓存/HTTP 下游慢；连接池指标、慢查询、依赖健康
  - 重试放大：多层重试叠加导致雪崩；统一重试与指数退避
- 命令与指标：`jstack` 看 `BLOCKED`/`WAITING`；`ss -tanp` 连接状态；Hikari 指标（活跃/空闲/等待）
- 修复路径：合理超时与重试、降级（本地缓存/默认值）

## 死锁与阻塞
- `jstack` 自动检测：`Found one Java-level deadlock`
- 检查 `synchronized`/`ReentrantLock`、数据库事务锁等待
- 修复：减少锁粒度、统一锁顺序、超时退出、异步化

## 资源耗尽（FD/线程/磁盘）
- FD：`lsof -p <pid> | wc -l`、`ulimit -n`；确保连接/文件关闭、日志轮转
- 线程：控制线程池最大值、栈大小 `-Xss` 与宿主限制；避免每请求新线程
- 磁盘/IO：日志刷盘与临时目录耗尽；压缩日志、异步写、迁移持久卷

## 网络与安全
- DNS/TLS：证书过期、协议不匹配；`dig`/TLS 握手失败
- 内核队列/错误：`netstat -s`、`sar -n DEV 1`；调整 `backlog`、连接复用

## 配置与类路径
- 配置错误/缺失：区分环境变量/配置中心；回滚配置；敏感项走 `Secret`
- 类冲突：依赖版本不兼容、重复类；`jdeps`、`mvn dependency:tree`

## 容器/Kubernetes 特定
- OOMKill vs OOME：检查 `requests/limits` 与 `kubectl describe pod` 事件
- 探针与重启：不当 `liveness` 触发雪崩；放宽 `startupProbe`
- CrashLoopBackOff：查看 `kubectl logs`/`describe`，排查镜像拉取/权限/挂载
- HPA 与冷启动：预热副本、限制扩容速度，避免抖动

## Arthas 实战用法
- 快速附着与会话管理：
  - 本机附着：`curl -O https://arthas.aliyun.com/arthas-boot.jar` → `java -jar arthas-boot.jar`（交互选择 `pid`）或 `java -jar arthas-boot.jar <pid>` 直接附着
  - 会话端口：附着后可用 `telnet 127.0.0.1 <port>` 或访问 `http://127.0.0.1:<port>/` 控制台
  - 退出与卸载：`quit` 仅退出当前会话；`stop`/`shutdown` 卸载 Arthas，不影响目标进程运行
  - 前置条件：优先使用 JDK 运行 `arthas-boot.jar`（依赖 `com.sun.tools.attach`）；同机同用户或具备足够权限；受限环境可能禁用 `attach`
  - 容器/K8s：在容器内执行上述命令；如需远程访问，暴露 telnet/http 端口（仅临时诊断，注意安全）

- 安装与附着：
  - 下载启动器：`curl -O https://arthas.aliyun.com/arthas-boot.jar`
  - 启动并选择进程：`java -jar arthas-boot.jar`（交互选择 `pid`）
  - 远程/容器：在容器内执行或通过调试端口与 Sidecar 方式注入
- 快速总览：
  - `dashboard` 查看线程、内存、GC、系统负载的即时面板
  - `thread` 查看线程信息、`thread -n 5` 列出最忙线程、`thread -b` 查找死锁
  - `jvm`/`sysprop`/`sysenv` 查看 JVM/系统属性与环境变量
- 热点与性能：
  - 方法级性能监控：`monitor -c 5 <Class> <method>`（每 5 次调用聚合统计）
  - 调用链与耗时：`trace <Class> <method> '#cost>100'`（只输出耗时 >100ms）
  - 观察入参/返回值：`watch <Class> <method> '{params, returnObj}' -x 2 -n 3`（深度 2，采样 3 次）
  - 栈追踪：`stack <Class> <method>`（打印当前调用栈）
- 代码与类加载：
  - 反编译：`jad <Class>`（确认实际生效代码与字节码）
  - 即时修补：`mc` 编译、`retransform` 重转换字节码（谨慎使用）
  - 搜索类与方法：`sc -d <ClassNamePattern>`、`sm <Class> <method>`
- 资源与内存：
  - 堆转储：`heapdump /tmp/heap.hprof`（生成堆 dump 供离线分析）
  - 对象统计：`ognl '@java.lang.management.ManagementFactory@getMemoryMXBean()'` 查询内存信息
- 连接池/线程池示例：
  - 监控 Hikari 活跃连接：`watch com.zaxxer.hikari.pool.HikariPool getActiveConnections '{returnObj}' -x 1 -n 5`
  - 监控线程池队列长度：`watch java.util.concurrent.ThreadPoolExecutor getQueue '{returnObj.size()}' -x 1 -n 5`
- 日志与参数：
  - 动态调整日志级别：`logger`（列出）→ `logger -c <name> -l debug`
  - JVM 选项：`vmoption`（查看/设置部分可动态参数）
- 典型排查流程：
  - 高 CPU：`dashboard` → `thread -n 5` → `profiler start --event cpu`（若已集成）→ `profiler stop` 生成报告
  - 慢接口：`trace <Controller> <method>` → `watch` 查看慢点入参与返回 → 针对依赖调用设置阈值定位热点
  - 内存膨胀：`dashboard`/`jvm` 查看堆占用 → `heapdump` 导出 → MAT 分析 Dominators
  - 类冲突：`sc -d` 查看加载来源 → `jad` 确认版本 → 依赖整合与重启验证

## 回滚与缓解
- 快速回滚：保障 `rollout undo`/版本切换；检查数据兼容与脚本
- 灰度与限流：逐步恢复流量；限制低优先任务
- 保护：熔断关键依赖、降级静态页/缓存；限重试与队列深度

## 命令速查
- 线程热点：`top -H -p <pid>` → `printf '%x\n' <tid>` → `jstack <pid>`
- 堆/GC：`jcmd <pid> GC.heap_info`、`jstat -gcutil <pid> 1s 10`
- 堆 dump：`jcmd <pid> GC.heap_dump filename=/tmp/heap.hprof` 或 `heapdump /tmp/heap.hprof`
- JFR：`jcmd <pid> JFR.start name=on_prod duration=60s filename=/tmp/on_prod.jfr`
- Arthas：`dashboard`、`thread -n 5`、`trace <Class> <method>`、`watch <Class> <method> '{params, returnObj}' -x 2`、`jad <Class>`、`heapdump`

## 发布前检查
- 启用 `HeapDumpOnOutOfMemoryError` 与 GC 日志；指标暴露与告警规则
- 线程池/连接池容量与超时、重试与限流策略一致
- 健康检查与探针参数合理；配置与密钥来源正确
- 压测与回滚预案、灰度策略与监控阈值校准
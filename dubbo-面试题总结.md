# Dubbo 面试题总结（Markdown 版）

> 覆盖架构与组件、协议与序列化、注册与发现、集群与负载均衡、路由与治理、SPI 扩展、配置与调优、可观测与排查、Dubbo3 特性、与 Spring Cloud/gRPC 对比、常见场景题与动手题。

## 目录
- 基础架构与调用链路
- 注册中心与服务发现
- 协议、序列化与传输
- 集群与负载均衡
- 路由与服务治理
- 配置项与运行参数
- SPI 扩展机制
- 可观测性与故障排查
- 性能优化与稳定性实践
- Dubbo3 新特性与演进
- 与 Spring Cloud/gRPC 的对比
- 高频面试题（含要点）
- 实战动手题

---

## 基础架构与调用链路
- 角色与组件：Provider、Consumer、Registry（Zookeeper/Nacos）、Config Center、Metadata Center、Monitor。
- 调用链路（简化）：Consumer 通过 `Proxy → Invoker → ClusterInvoker → Directory → Router → LoadBalance → Client` 选择 Provider 并发起请求，Provider 端 `Server → Filter → Biz` 处理。
- 生命周期：服务暴露（export）、引用（refer）、注册与订阅（register/subscribe）、上线/下线事件。
- 线程模型：服务端业务线程池隔离（固定/缓存/限流）；网络 IO 与业务线程分离。

## 注册中心与服务发现
- Zookeeper：临时节点存活与会话、Watch 订阅；注册表结构（`/dubbo/com.xxx.Service/providers`）。
- Nacos：Naming 与 Config 合并；健康检查、权重与元数据；服务变更推送。
- 直连与多注册中心：支持不经注册中心直连；多中心聚合与优先级、分区容错。
- 元数据中心：方法级参数、路由标签、版本信息的分发与治理。

## 协议、序列化与传输
- Dubbo 协议：基于长连接、NIO、请求响应、单一连接复用；适合短小频繁 RPC。
- Triple（Dubbo3）：基于 gRPC/HTTP2，支持多语言互通、流式传输与网格化。
- 其他协议：RMI、REST、Hessian、gRPC；选择依据为生态、跨语言、性能与网络环境。
- 序列化：Hessian2/Kryo/FST/Protobuf/JSON；选择依据为性能、兼容性、演变与调试友好。

### Dubbo 的序列化方式与默认值
- Dubbo 协议默认使用 `Hessian2`，可切换为 `Kryo`、`FST`、`FastJson`、`Java`（不推荐）等。
- Dubbo3 的 `Triple` 协议默认使用 `Protobuf`（跨语言强类型），也可使用 `JSON` 作为兼容与调试选择。
- RMI 协议通常使用 Java 原生序列化；REST 以 `JSON` 为主；gRPC 使用 `Protobuf`。
- 选型原则：
  - 需要跨语言与强类型：`Protobuf`（Triple/gRPC）
  - Java 内部高性能：`Kryo`/`FST`（注意对象演进与兼容策略）
  - 调试友好与快速集成：`JSON`（带来体积与性能成本）
  - 遗留与兼容：`Hessian2` 在 Dubbo 生态成熟，兼容性较好

### 配置示例
- XML 配置切换序列化：
```xml
<dubbo:protocol name="dubbo" serialization="hessian2" port="20880"/>
<dubbo:protocol name="dubbo" serialization="kryo" port="20880"/>
<dubbo:protocol name="tri" serialization="protobuf" port="50051"/>
```
- Spring Boot（application.yml）：
```yaml
dubbo:
  protocol:
    name: dubbo
    port: 20880
    serialization: hessian2
```
- 也可在 `provider`/`consumer` 层级指定，覆盖协议层默认设置；确保两端协商一致。

### 序列化对比与选型建议
- Hessian2
  - 优点：Java 生态成熟、支持复杂对象与多态、无需预先定义 schema，兼容性好于 Java 原生；性能与体积均优于 JSON
  - 局限：跨语言支持有限；演进时字段名修改需注意反序列化兼容
  - 适用：Dubbo 协议默认选择，内部服务兼容性优先场景
- Kryo
  - 优点：高性能、体积小、支持对象图与循环引用；可注册类提升性能与稳定性
  - 局限：Java 友好为主，跨语言弱；类演进需谨慎，字段顺序与版本兼容要管理；安全需做反序列化白名单
  - 适用：纯 Java 内部高吞吐、低延迟场景
- FST
  - 优点：与 Kryo 类似的高性能与高压缩率，API 简洁
  - 局限：社区活跃度与长期维护需评估；兼容与安全策略同 Kryo
  - 适用：纯 Java 性能优先场景的替代方案
- Protobuf
  - 优点：跨语言、体积小、速度快、强类型；严格的字段编号与演进规则（保留字段、不可复用编号）
  - 局限：不直接支持多态与复杂对象图，需通过 oneof/枚举等建模；需要 .proto 与代码生成
  - 适用：Dubbo3 Triple/gRPC、跨语言通信、对外接口与强演进约束场景
- JSON（FastJson/fastjson2 等）
  - 优点：人类可读、调试友好、松耦合；字段增删容忍度高
  - 局限：体积大、性能较低；类型信息不明确导致歧义；需禁用不安全特性（如 autoType）
  - 适用：外部接口快速兼容、调试与日志、对性能不敏感场景
- Java 原生序列化
  - 优点：零改造可用（遗留）
  - 局限：慢、体积大、安全风险高（反序列化链）、跨版本兼容差；生产不推荐
  - 适用：仅限遗留与过渡，不建议新项目使用

- 选择维度与建议
  - 跨语言：Protobuf 明显优于其他；JSON 次之；Kryo/FST/Hessian2 偏 Java
  - 性能与体积：Kryo/FST ≈ Protobuf 优于 Hessian2 优于 JSON；Java 原生最差
  - 可演进性：Protobuf 有严格演进约束但可控；JSON 最松但易歧义；Kryo/FST 需类注册与版本管理；Hessian2 中等
  - 调试与可读性：JSON 最优；其他为二进制需工具支持
  - 安全：避免 Java 原生；为 Kryo/FST/Hessian2 配置反序列化白名单与限速；JSON 禁用危险特性

## 集群与负载均衡
- Cluster 容错策略：
  - Failover（失败重试，适合读操作）；Failfast（快速失败，适合写操作）；Failsafe（吞掉异常用于通知类）；Failback（记录失败异步重发）；Forking（并发请求返回最快结果）；Broadcast（广播执行、适合通知）。
- LoadBalance 算法：
  - Random（权重随机）、RoundRobin（加权轮询）、LeastActive（最少活跃数优先）、ConsistentHash（一致性哈希、同键路由到同节点）。
- Sticky 与连接复用：粘滞调用保持会话亲和；减少反复切换带来的缓存缺失。

## 路由与服务治理
- 路由器：条件路由（Condition）、脚本路由（Script）、Tag 路由（灰度与分区）；支持黑白名单、权重与版本优先级。
- 限流与熔断：线程池/队列容量、异常比例、响应时间阈值；与 Sentinel/Resilience4j 联动。
- 服务降级与 Mock：在 Provider 不可用时返回默认值或回退逻辑；方法级 mock 配置。
- 灰度发布：按 Tag/用户/比例路由；回滚与探活策略。

## 配置项与运行参数
- 连接与超时：`timeout`、`retries`、`connections`、`actives`、`lazy`（延迟连接）、`sticky`。
- 线程池：`threadpool`（fixed/cached/limited）、`threads`、`queues`、`reject` 策略。
- 版本与分组：`version`、`group` 便于兼容与多租户隔离。
- 注册与协议：`registry`、多注册中心；`protocol` 选择与端口、序列化方式。
- 方法级配置：`methods` 节点可配置方法级 `timeout/retries/loadbalance`。

## SPI 扩展机制
- Dubbo SPI 与 JDK SPI 差异：支持 `@SPI`、`@Adaptive`、`@Activate`，可按 URL 参数动态选择实现，支持 Wrapper 装饰链。
- 扩展点加载：`ExtensionLoader` 解析资源文件 `META-INF/dubbo/`、`META-INF/services/`；支持别名与优先级。
- Adaptive 扩展：根据运行时 URL 参数决定实现（如 `protocol`/`serialization`）。
- Filter/Router/Cluster/LoadBalance 扩展：按条件自动激活或指定激活顺序。

## 可观测性与故障排查
- 调用统计：TPS、RT 分位数、错误率、重试次数；方法级与节点级维度。
- 链路追踪：OpenTelemetry/SkyWalking，染色与跨线程传播；识别下游瓶颈与重试放大。
- 常见问题：
  - 时延飙升：核查线程池饱和、下游慢、序列化开销、GC 与网络。
  - 重试风暴：`retries` 过高、超时不合理；统一重试策略与指数退避。
  - 路由失效：规则未下发或匹配错误；查看元数据与本地缓存。
  - 注册异常：ZK 会话失效、Nacos 权限或心跳失败；检查日志与连接状态。

## 性能优化与稳定性实践
- 连接与线程：按接口拆分线程池与限流；复用连接与批量请求；避免阻塞调用嵌套。
- 序列化优化：选择 Protobuf/Kryo 等高效方案；控制对象层级与大小；避免循环引用与大集合。
- 数据设计：DTO 稳定性与版本演进；压缩与差异化传输；分页与流式。
- 配置治理：方法级精细化 `timeout/retries`；容错策略分场景；灰度路由与动态权重。

## Dubbo3 新特性与演进
- Triple 协议：HTTP/2 + gRPC 互通，多语言支持，网格与服务发现更友好。
- Mesh 集成：与 Istio 等服务网格协同，使用 xDS/Sidecar 路由与治理。
- 应用级服务发现：跨注册中心与跨集群治理；元数据增强。
- 多语言与跨平台：IDL 与 Protobuf 支持增强；Gateway/HTTP 入口能力提升。

## 版本里程碑与发布时间（节选）
- 3.3（正式版）：2024-09-11（Triple X 升级）
  - 参考：https://dubbo.apache.org/en/blog/2024/09/11/apache-dubbo-3.3-released-triple-x-leads-a-new-era-of-microservices-communication/
- 3.2.0-beta.4（功能版）：2023-01-27
  - 参考：https://dubbo.apache.org/en/blog/2023/01/30/dubbo-3.1.5-and-3.2.0-beta.4-official-release/
- 3.1.5（稳定版）：2023-01-21
  - 参考（含完整列表）：https://dubbo.apache.org/en/release/past-releases/java/
- 3.0.13：2023-01-27
  - 参考：同上“Past Releases/Java SDK”
- 3.0.1（3.0 线早期稳定公告）：2021-07-02
  - 参考：https://cn.dubbo.apache.org/en/blog/2021/07/02/dubbo-java-3.0.1-release-announcement/
- 2.7.21（长期维护线）：2023-01-21
  - 参考：Past Releases 页面
- 2.7.20（长期维护线）：2022-12-30
  - 参考：Past Releases 页面
- 2.7.19（长期维护线）：2022-12-13
  - 参考：Past Releases 页面
- 2.6.x：历史线，存量维护（具体版本与日期见 GitHub Releases）
  - 参考：https://github.com/apache/dubbo/releases

说明：以上为常见里程碑与近年发布时间，完整与最新的版本与日期请以官方“Past Releases/Java SDK”与 GitHub Releases 为准。

## 与 Spring Cloud/gRPC 的对比
- Spring Cloud：生态丰富、HTTP 友好、配置与治理集中；RPC 性能不及长连接协议；适合微服务全家桶。
- gRPC：强类型、跨语言、HTTP/2 流控与双向流；需要网关与生态集成；与 Dubbo3 Triple 互通。
- Dubbo：Java 生态强、RPC 经验成熟、治理能力细粒度；跨语言能力在 Dubbo3 中增强。

## 高频面试题（含要点）
- Dubbo 的调用链路与各环节职责？
  - Proxy/Invoker/Cluster/Directory/Router/LoadBalance；Provider 端的 Filter 与线程池隔离。
- 如何选择负载均衡与容错策略？
  - 读场景用 Failover+LeastActive；写场景用 Failfast+Random；同键一致性用 ConsistentHash。
- 如何做灰度发布与路由？
  - Tag/条件/脚本路由；按版本、用户、比例；确保规则下发一致与回滚机制。
- 重试导致雪崩如何治理？
  - 统一超时与重试策略、指数退避、舱壁隔离；避免多层重试叠加。
- Dubbo SPI 的 Adaptive 原理？
  - 根据 URL 参数生成代理在运行时选择实现；`@Adaptive` 指定参数键。
- 注册中心挂掉后会怎样？
  - 已订阅的地址仍可用（本地缓存）；新发布/下线无法感知；重连后恢复。
- 序列化如何选型？
  - 兼顾性能与演进：跨语言用 Protobuf/Triple；Java 内部可用 Hessian2/Kryo。
- 如何排查 RT 飙升？
  - 指标与链路定位→线程池与下游→序列化与 GC→网络与路由规则。
- Dubbo3 Triple 有何优势？
  - HTTP/2 多路复用、跨语言、与网格生态兼容。

## 实战动手题
- 基于 Tag 路由实现灰度发布：
  - 规则：`用户ID % 10 < 2` 走新版本；观察 RT/错误率，失败快速回滚。
- 方法级容错与重试策略：
  - 为读接口配置 `retries=2 timeout=200ms`，写接口 `retries=0 failfast`；压测验证。
- SPI 扩展 Filter：
  - 编写 `Filter` 记录方法耗时与异常分类；按 `@Activate(order=...)` 控制顺序。
- Triple 协议与 gRPC 互通：
  - 定义 Protobuf IDL，分别用 Dubbo3 与 gRPC 启动服务并互调，验证跨语言兼容与性能差异。

---

如果你需要我按你的项目栈（Zookeeper/Nacos、Dubbo2/Dubbo3、业务场景）生成“逐题参考答案”或“运行参数调优清单”，告诉我偏好与深度，我可以继续补充。

---

## 使用示例（Spring Boot，Provider/Consumer）

### 依赖（Maven）
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>dubbo-sample</artifactId>
  <version>1.0.0</version>
  <properties>
    <java.version>17</java.version>
    <spring.boot.version>3.3.0</spring.boot.version>
    <dubbo.version>3.3.0</dubbo.version>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <version>${spring.boot.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
      <version>${spring.boot.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.dubbo</groupId>
      <artifactId>dubbo-spring-boot-starter</artifactId>
      <version>${dubbo.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.dubbo</groupId>
      <artifactId>dubbo-registry-zookeeper</artifactId>
      <version>${dubbo.version}</version>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <version>${spring.boot.version}</version>
      </plugin>
    </plugins>
  </build>
</project>
```

### 公共接口
```java
package com.example.api;

public interface GreetingService {
    String greet(String name);
}
```

### Provider
```java
package com.example.provider;

import org.apache.dubbo.config.annotation.DubboService;
import com.example.api.GreetingService;

@DubboService
public class GreetingServiceImpl implements GreetingService {
    @Override
    public String greet(String name) {
        return "Hello, " + name;
    }
}
```

```java
package com.example.provider;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProviderApplication.class, args);
    }
}
```

```yaml
# application.yml（Provider，Dubbo 协议 + Zookeeper）
server:
  port: 8081
dubbo:
  application:
    name: dubbo-provider
  protocol:
    name: dubbo
    port: 20880
  registry:
    address: zookeeper://127.0.0.1:2181
```

### Consumer
```java
package com.example.consumer;

import org.apache.dubbo.config.annotation.DubboReference;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import com.example.api.GreetingService;

@RestController
public class GreetingController {
    @DubboReference
    private GreetingService greetingService;

    @GetMapping("/greet")
    public String greet(@RequestParam String name) {
        return greetingService.greet(name);
    }
}
```

```java
package com.example.consumer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }
}
```

```yaml
# application.yml（Consumer，Dubbo 协议 + Zookeeper）
server:
  port: 8080
dubbo:
  application:
    name: dubbo-consumer
  registry:
    address: zookeeper://127.0.0.1:2181
```

### Dubbo3 Triple（gRPC/HTTP2）示例配置
```yaml
# Provider 切换为 Triple 协议
dubbo:
  application:
    name: dubbo-provider
  protocol:
    name: tri
    port: 50051
  registry:
    address: zookeeper://127.0.0.1:2181
```

```yaml
# Consumer 保持不变，仅需确保依赖与接口一致
dubbo:
  application:
    name: dubbo-consumer
  registry:
    address: zookeeper://127.0.0.1:2181
```

### 使用 Nacos（可选）
```yaml
dubbo:
  application:
    name: dubbo-provider
  protocol:
    name: dubbo
    port: 20880
  registry:
    address: nacos://127.0.0.1:8848
```

### 启动与调用
- 启动 Provider 与 Consumer
- 访问 `http://localhost:8080/greet?name=Dubbo`
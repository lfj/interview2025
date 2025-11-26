# Spring Boot 面试题总结（Markdown 版）

> 覆盖基础特性、自动配置原理、Starter 机制、启动流程、配置管理、监控管理、测试、部署打包、性能优化、故障排查、最佳实践。高频考点与追问路径，含示例代码与配置。

## 目录
- [基础与特性](#基础与特性)
- [自动配置原理](#自动配置原理)
- [Starter 机制](#starter-机制)
- [配置文件与属性绑定](#配置文件与属性绑定)
- [启动流程与生命周期](#启动流程与生命周期)
- [Web 开发](#web-开发)
- [数据访问](#数据访问)
- [监控与管理（Actuator）](#监控与管理actuator)
- [测试](#测试)
- [部署与打包](#部署与打包)
- [性能优化](#性能优化)
- [故障排查](#故障排查)
- [最佳实践](#最佳实践)
- [常见场景题](#常见场景题)
- [实战配置题](#实战配置题)

---

## 基础与特性

1. **Spring Boot 核心特性**
   - 要点：约定优于配置、自动配置、起步依赖（Starter）、内嵌服务器、生产就绪特性。
   - 追问：为什么选择 Spring Boot；与 Spring Framework 的关系。

2. **Spring Boot vs Spring MVC**
   - 要点：Spring Boot 是 Spring 的扩展，简化配置；内嵌 Tomcat/Jetty；零配置启动。
   - 追问：如何从 Spring MVC 迁移到 Spring Boot；兼容性考虑。

3. **Spring Boot 版本选择**
   - 要点：关注 LTS 版本；JDK 版本兼容性；依赖管理（BOM）。
   - 追问：如何升级 Spring Boot 版本；破坏性变更处理。

4. **内嵌服务器**
   - 要点：Tomcat、Jetty、Undertow；如何切换；性能对比。
   - 追问：为什么使用内嵌服务器；生产环境部署方式。

### 内嵌服务器详解

#### 三种服务器对比

**Tomcat（默认）**
- 特点：Apache 基金会项目，最成熟稳定，使用最广泛
- 优势：生态丰富、文档完善、社区支持好、兼容性强
- 劣势：内存占用相对较大、性能中等
- 适用场景：大多数 Web 应用，特别是需要稳定性的生产环境

**Jetty**
- 特点：Eclipse 基金会项目，轻量级、嵌入式友好
- 优势：启动快、内存占用小、支持异步 Servlet、适合嵌入式应用
- 劣势：性能略低于 Undertow，社区相对较小
- 适用场景：微服务、嵌入式应用、需要快速启动的场景

**Undertow**
- 特点：Red Hat 开发，基于 NIO，高性能
- 优势：性能最好、内存占用最小、支持 HTTP/2、WebSocket 性能优秀
- 劣势：相对较新，生态不如 Tomcat 丰富
- 适用场景：高并发、低延迟要求的应用，需要极致性能的场景

#### 性能对比（基准测试参考）

| 指标 | Tomcat | Jetty | Undertow |
|------|--------|-------|----------|
| **吞吐量** | 中等 | 中等 | **最高** |
| **内存占用** | 较高 | 中等 | **最低** |
| **启动速度** | 中等 | **最快** | 快 |
| **并发处理** | 良好 | 良好 | **优秀** |
| **HTTP/2 支持** | 支持 | 支持 | **原生支持** |
| **WebSocket** | 支持 | 支持 | **性能最优** |
| **稳定性** | **最稳定** | 稳定 | 稳定 |
| **生态成熟度** | **最成熟** | 成熟 | 较新 |

#### 如何切换内嵌服务器

**方式一：Maven 排除 + 引入（推荐）**

切换到 Jetty：
```xml
<dependencies>
    <!-- 排除 Tomcat -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    
    <!-- 引入 Jetty -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jetty</artifactId>
    </dependency>
</dependencies>
```

切换到 Undertow：
```xml
<dependencies>
    <!-- 排除 Tomcat -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    
    <!-- 引入 Undertow -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-undertow</artifactId>
    </dependency>
</dependencies>
```

**方式二：Gradle 配置**

切换到 Undertow：
```gradle
dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web') {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-tomcat'
    }
    implementation 'org.springframework.boot:spring-boot-starter-undertow'
}
```

**方式三：使用专用 Starter**

直接使用对应的 starter（如果项目结构支持）：
```xml
<!-- Jetty Starter -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>

<!-- Undertow Starter -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

#### 服务器配置示例

**Tomcat 配置**
```yaml
server:
  tomcat:
    threads:
      max: 200
      min-spare: 10
    max-connections: 10000
    accept-count: 100
    connection-timeout: 20000
    compression: on
    compression-min-size: 2048
    compressable-mime-types: text/html,text/xml,text/plain,text/css,text/javascript,application/javascript,application/json
```

**Jetty 配置**
```yaml
server:
  jetty:
    threads:
      max: 200
      min: 10
      idle-timeout: 60000
    connection-idle-timeout: 30000
    acceptors: -1  # -1 表示自动计算
    selectors: -1
```

**Undertow 配置**
```yaml
server:
  undertow:
    threads:
      io: 4
      worker: 64
    buffer-size: 1024
    direct-buffers: true
    max-connections: 10000
    max-http-post-size: 10485760
```

#### 性能调优建议

**Tomcat 调优**
- 调整线程池大小：`server.tomcat.threads.max`
- 启用压缩：`server.tomcat.compression=on`
- 调整连接超时：`server.tomcat.connection-timeout`
- 使用 NIO2 连接器（默认）

**Jetty 调优**
- 调整线程池：`server.jetty.threads.max`
- 配置连接器：`server.jetty.acceptors`、`server.jetty.selectors`
- 启用 HTTP/2：需要额外配置

**Undertow 调优**
- 调整 IO 线程：`server.undertow.threads.io`（通常为 CPU 核心数）
- 调整工作线程：`server.undertow.threads.worker`（通常为 CPU 核心数 × 8）
- 启用直接内存：`server.undertow.direct-buffers=true`
- 调整缓冲区大小：`server.undertow.buffer-size`

#### 选择建议

**选择 Tomcat 如果：**
- 需要最稳定的生产环境
- 团队熟悉 Tomcat
- 需要丰富的文档和社区支持
- 应用对性能要求不是极致

**选择 Jetty 如果：**
- 需要快速启动（如微服务）
- 内存资源有限
- 需要嵌入式部署
- 需要异步 Servlet 特性

**选择 Undertow 如果：**
- 需要极致性能（高并发、低延迟）
- 需要 HTTP/2 原生支持
- 需要优秀的 WebSocket 性能
- 内存资源紧张
- 对性能要求高于稳定性要求

#### 实际性能测试示例

```java
// 使用 JMeter 或 wrk 进行压测
// 示例：wrk 压测命令
// wrk -t12 -c400 -d30s --latency http://localhost:8080/api/test

// 典型结果（仅供参考，实际结果因环境而异）：
// Tomcat:  ~15,000 req/s, 平均延迟 25ms
// Jetty:   ~16,000 req/s, 平均延迟 23ms  
// Undertow: ~20,000 req/s, 平均延迟 18ms
```

#### 注意事项

1. **兼容性**：大多数情况下三种服务器可以无缝切换，但某些高级特性可能有差异
2. **监控**：切换后需要验证 Actuator 端点是否正常工作
3. **生产验证**：切换前应在测试环境充分验证
4. **依赖冲突**：确保完全排除旧服务器依赖，避免冲突
5. **配置迁移**：不同服务器的配置项名称不同，需要相应调整

5. **Spring Boot CLI**
   - 要点：命令行工具；Groovy 脚本；快速原型开发。
   - 追问：适用场景；与 Maven/Gradle 对比。

---

## 自动配置原理

1. **自动配置机制**
   - 要点：`@EnableAutoConfiguration`；`spring.factories`；条件注解（`@ConditionalOnClass`、`@ConditionalOnProperty`）。
   - 追问：自动配置的执行顺序；如何排除自动配置。

2. **条件注解详解**
   - 要点：`@ConditionalOnClass`、`@ConditionalOnBean`、`@ConditionalOnProperty`、`@ConditionalOnMissingBean`。
   - 追问：条件注解的执行时机；如何自定义条件注解。

3. **自动配置类加载流程**
   - 要点：`SpringFactoriesLoader.loadFactoryNames()`；`@EnableAutoConfiguration` 导入 `AutoConfigurationImportSelector`。
   - 追问：为什么使用 `spring.factories`；如何调试自动配置。

4. **自动配置优先级**
   - 要点：用户配置 > 自动配置；`@Order` 注解；`application.properties` 优先级。
   - 追问：如何覆盖自动配置；配置冲突解决。

### 示例：自动配置类
```java
@Configuration
@ConditionalOnClass(DataSource.class)
@ConditionalOnProperty(prefix = "spring.datasource", name = "url")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

### 示例：排除自动配置
```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    RedisAutoConfiguration.class
})
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## Starter 机制

1. **Starter 的作用**
   - 要点：依赖管理；自动配置；简化依赖声明。
   - 追问：为什么需要 Starter；如何创建自定义 Starter。

2. **常用 Starter**
   - 要点：`spring-boot-starter-web`、`spring-boot-starter-data-jpa`、`spring-boot-starter-redis`、`spring-boot-starter-test`。
   - 追问：Starter 命名规范；Starter 依赖传递。

3. **自定义 Starter 开发**
   - 要点：创建 `autoconfigure` 模块；编写自动配置类；创建 `spring.factories`。
   - 追问：Starter 版本管理；向后兼容性。

### 示例：自定义 Starter
```java
// 1. 自动配置类
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyProperties properties) {
        return new MyService(properties);
    }
}

// 2. spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration
```

---

## 配置文件与属性绑定

1. **配置文件优先级**
   - 要点：`application.properties`、`application.yml`；profile 配置（`application-{profile}.yml`）；外部配置优先级。
   - 追问：配置加载顺序；如何覆盖配置。

2. **@Value vs @ConfigurationProperties**
   - 要点：`@Value` 单个属性；`@ConfigurationProperties` 批量绑定；类型安全。
   - 追问：何时使用哪种方式；属性验证。

3. **Profile 机制**
   - 要点：`spring.profiles.active`；多环境配置；`@Profile` 注解。
   - 追问：如何切换环境；Profile 继承。

4. **外部化配置**
   - 要点：命令行参数、环境变量、`@PropertySource`、配置中心。
   - 追问：配置优先级；敏感信息处理。

5. **配置属性验证**
   - 要点：`@Validated`、`@NotNull`、`@Min`、`@Max`；自定义验证器。
   - 追问：验证失败处理；启动时验证。

### 示例：@ConfigurationProperties
```java
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {
    
    @NotBlank
    private String name;
    
    @Min(1)
    @Max(100)
    private int maxConnections;
    
    private List<String> servers = new ArrayList<>();
    
    // getters and setters
}

// application.yml
app:
  name: my-app
  max-connections: 50
  servers:
    - server1.example.com
    - server2.example.com
```

### 示例：Profile 配置
```yaml
# application.yml
spring:
  profiles:
    active: dev

---
# application-dev.yml
server:
  port: 8080
logging:
  level:
    root: DEBUG

---
# application-prod.yml
server:
  port: 80
logging:
  level:
    root: INFO
```

---

## 启动流程与生命周期

1. **SpringApplication 启动流程**
   - 要点：创建 `ApplicationContext`；加载 Bean；执行 `CommandLineRunner`/`ApplicationRunner`。
   - 追问：启动事件；自定义启动逻辑。

2. **ApplicationContext 初始化**
   - 要点：`AnnotationConfigApplicationContext`；Bean 扫描；Bean 创建。
   - 追问：启动性能优化；懒加载策略。

3. **启动事件**
   - 要点：`ApplicationStartingEvent`、`ApplicationReadyEvent`、`ApplicationFailedEvent`。
   - 追问：事件监听器；异步事件处理。

4. **CommandLineRunner vs ApplicationRunner**
   - 要点：两者在启动后执行；参数类型不同（String[] vs ApplicationArguments）。
   - 追问：执行顺序；异常处理。

5. **Banner（启动横幅）**
   - 要点：启动时显示在控制台的 ASCII 艺术字；默认显示 "SPRING BOOT"；可自定义或关闭。
   - 追问：如何自定义 Banner；Banner 显示时机；性能影响。

6. **优雅关闭**
   - 要点：`spring.lifecycle.timeout-per-shutdown-phase`；`@PreDestroy`；`ApplicationListener<ContextClosedEvent>`。
   - 追问：如何实现优雅关闭；超时处理。

### 示例：启动监听器
```java
@Component
public class MyApplicationListener implements ApplicationListener<ApplicationReadyEvent> {
    
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        System.out.println("Application is ready!");
        // 初始化逻辑
    }
}

@Component
public class MyCommandLineRunner implements CommandLineRunner {
    
    @Override
    public void run(String... args) {
        System.out.println("Application started with args: " + Arrays.toString(args));
    }
}
```

### Banner（启动横幅）详解

#### 什么是 Banner

Banner 是 Spring Boot 应用启动时在控制台显示的 ASCII 艺术字，默认显示 "SPRING BOOT" 字样。它出现在应用启动的最开始，用于标识应用和展示品牌信息。

**默认 Banner 示例：**
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.7.0)
```

#### Banner 的作用

Banner 在 Spring Boot 应用中具有多重作用，既有实用价值，也有展示价值：

**1. 品牌展示与识别**
- **应用标识**：在启动日志中清晰显示应用名称，便于在多服务环境中快速识别
- **版本信息**：展示应用版本号，方便运维人员确认部署版本
- **团队标识**：可以展示公司/团队 Logo 或名称，增强品牌识别度

**实际场景：**
```
# 多服务启动时，通过 Banner 快速识别
[Service-A] 启动中...
[Service-B] 启动中...
[Service-C] 启动中...
```

**2. 启动状态标识**
- **启动确认**：Banner 出现表示应用开始启动，便于监控和调试
- **快速定位**：在大量日志中，Banner 作为明显的分隔符，快速定位启动日志
- **启动时间点**：Banner 显示时机可以标记应用启动的起始时间

**实际场景：**
```bash
# 日志中看到 Banner，说明应用开始启动
2024-01-15 10:00:00.000  INFO --- [main] Application : Starting Application
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.7.0)
2024-01-15 10:00:05.123  INFO --- [main] Application : Started Application
```

**3. 环境信息展示**
- **环境标识**：显示当前运行环境（dev/test/prod），避免环境混淆
- **配置信息**：展示关键配置参数（如数据库地址、Redis 地址等）
- **系统信息**：显示 Java 版本、Spring Boot 版本等系统级信息

**实际示例：**
```
  _____ _                 _     ____             _   
 |_   _| |__   ___ _ __  | | __| __ )  ___  ___| |_ 
   | | | '_ \ / _ \ '_ \ | |/ /|  _ \ / _ \/ _ \ __|
   | | | | | |  __/ | | ||   < | |_) |  __/  __/ |_ 
   |_| |_| |_|\___|_| |_||_|\_\|____/ \___|\___|\__|
   
   Application: UserService
   Version: 1.2.3
   Environment: ${spring.profiles.active:dev}
   Spring Boot: ${spring-boot.version}
   Java: ${java.version}
   Build Time: 2024-01-15 10:00:00
```

**4. 个性化与团队文化**
- **视觉识别**：通过独特的 ASCII 艺术字增强视觉识别度
- **团队文化**：展示团队特色，增强归属感
- **趣味性**：在开发环境中增加趣味性，提升开发体验

**实际示例：**
```
    ___       __   __   __   __   __   __   __   __
   / __\     /  \ /  \ /  \ /  \ /  \ /  \ /  \ /  \
  / /       / /\ \ / /\ \ / /\ \ / /\ \ / /\ \ / /\ \
 / /___    / /  \ V /  \ V /  \ V /  \ V /  \ V /  \ V /
 \____/   /_/    \_/    \_/    \_/    \_/    \_/    \_/
 
   :: Awesome Team ::  Keep Coding, Keep Awesome!
```

**5. 调试与故障排查**
- **版本确认**：快速确认运行的应用版本，便于问题定位
- **环境确认**：确认当前运行环境，避免环境配置错误
- **启动顺序**：在微服务架构中，通过 Banner 确认服务启动顺序

**实际场景：**
```bash
# 故障排查时，通过 Banner 确认版本和环境
# 问题：生产环境出现 Bug
# 排查：查看启动日志中的 Banner，确认版本是否为最新版本
# 发现：Banner 显示版本为 1.0.0，但应该部署 1.1.0
# 结论：部署版本错误，导致 Bug
```

**6. 监控与告警集成**
- **启动监控**：监控系统可以通过 Banner 的出现判断应用是否启动
- **版本监控**：自动提取 Banner 中的版本信息，进行版本一致性检查
- **环境监控**：通过 Banner 中的环境信息，验证环境配置是否正确

**实际场景：**
```java
// 监控脚本可以通过 Banner 提取信息
// 启动日志 → 解析 Banner → 提取版本/环境 → 上报监控系统
```

**7. 开发体验提升**
- **启动反馈**：给开发者明确的启动反馈，提升开发体验
- **信息集中**：将关键信息集中在 Banner 中，一目了然
- **专业感**：自定义 Banner 让应用看起来更专业

**8. 安全与合规**
- **环境警告**：在非生产环境的 Banner 中显示警告信息
- **合规信息**：显示版权信息、许可证信息等
- **访问控制提示**：显示访问控制相关的提示信息

**实际示例（安全警告）：**
```
  ⚠️  WARNING: DEVELOPMENT ENVIRONMENT  ⚠️
  This is a development server.
  Do not use production data!
  
  Application: UserService
  Environment: dev
  Version: 1.2.3-SNAPSHOT
```

#### Banner 作用总结

| 作用类型 | 主要价值 | 适用场景 |
|---------|---------|---------|
| **品牌展示** | 增强识别度 | 多服务环境、微服务架构 |
| **启动标识** | 快速定位 | 日志分析、故障排查 |
| **信息展示** | 集中展示关键信息 | 版本管理、环境确认 |
| **个性化** | 提升体验 | 开发环境、团队文化 |
| **调试辅助** | 问题定位 | 故障排查、版本确认 |
| **监控集成** | 自动化监控 | CI/CD、运维监控 |
| **安全合规** | 安全提示 | 环境隔离、合规要求 |

#### 最佳实践建议

1. **开发环境**：使用个性化 Banner，增加开发乐趣
2. **测试环境**：显示环境标识和版本信息，便于测试验证
3. **生产环境**：使用简洁的 Banner，或关闭 Banner 以提升性能
4. **多环境区分**：不同环境使用不同的 Banner，避免环境混淆
5. **信息适度**：不要显示敏感信息（如密码、密钥等）
6. **性能考虑**：Banner 文件不要过大，避免影响启动速度

#### Banner 显示模式

Spring Boot 提供了三种 Banner 显示模式：

```java
public enum Mode {
    OFF,        // 关闭 Banner
    CONSOLE,    // 仅控制台显示（默认）
    LOG         // 仅日志文件显示
}
```

#### 如何控制 Banner

**方式一：配置文件（推荐）**
```yaml
# application.yml
spring:
  banner:
    location: classpath:banner.txt  # 自定义 Banner 文件路径
    charset: UTF-8                  # 字符编码
    image:
      location: classpath:banner.png # 图片 Banner（需要转换为 ASCII）
      bitdepth: 4                    # 位深度
      hightext: true                 # 高对比度文本
```

**方式二：代码配置**
```java
@SpringBootApplication
public class Application {
    
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        
        // 关闭 Banner
        app.setBannerMode(Banner.Mode.OFF);
        
        // 或者设置自定义 Banner
        app.setBanner(new CustomBanner());
        
        app.run(args);
    }
}
```

**方式三：环境变量**
```bash
# 关闭 Banner
export SPRING_BANNER_MODE=off

# 或启动参数
java -jar app.jar --spring.banner.mode=off
```

#### 自定义 Banner 文件

**步骤 1：创建 Banner 文件**

在 `src/main/resources/` 目录下创建 `banner.txt` 文件：

```
  _____ _                 _     ____             _   
 |_   _| |__   ___ _ __  | | __| __ )  ___  ___| |_ 
   | | | '_ \ / _ \ '_ \ | |/ /|  _ \ / _ \/ _ \ __|
   | | | | | |  __/ | | ||   < | |_) |  __/  __/ |_ 
   |_| |_| |_|\___|_| |_||_|\_\|____/ \___|\___|\__|
   
   :: My Application :: (v1.0.0)
```

**步骤 2：配置 Banner 文件路径**
```yaml
spring:
  banner:
    location: classpath:banner.txt
```

**步骤 3：使用变量（可选）**

Banner 支持 Spring Boot 的变量替换：

```
${application.name:MyApp}
${application.version:1.0.0}
${spring-boot.version}
${java.version}
${spring.profiles.active:default}
```

**示例 Banner 文件（带变量）：**
```
  _____ _                 _     ____             _   
 |_   _| |__   ___ _ __  | | __| __ )  ___  ___| |_ 
   | | | '_ \ / _ \ '_ \ | |/ /|  _ \ / _ \/ _ \ __|
   | | | | | |  __/ | | ||   < | |_) |  __/  __/ |_ 
   |_| |_| |_|\___|_| |_||_|\_\|____/ \___|\___|\__|
   
   Application: ${application.name:MyApp}
   Version: ${application.version:1.0.0}
   Spring Boot: ${spring-boot.version}
   Java: ${java.version}
   Profile: ${spring.profiles.active:default}
```

#### 自定义 Banner 类

实现 `Banner` 接口，完全自定义 Banner 内容：

```java
import org.springframework.boot.Banner;
import org.springframework.core.env.Environment;

import java.io.PrintStream;

public class CustomBanner implements Banner {
    
    @Override
    public void printBanner(Environment environment, 
                           Class<?> sourceClass, 
                           PrintStream out) {
        out.println();
        out.println("=================================");
        out.println("    My Spring Boot Application  ");
        out.println("=================================");
        out.println("Environment: " + 
                   environment.getProperty("spring.profiles.active", "default"));
        out.println("Version: " + 
                   environment.getProperty("application.version", "1.0.0"));
        out.println("=================================");
        out.println();
    }
}

// 使用自定义 Banner
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        app.setBanner(new CustomBanner());
        app.run(args);
    }
}
```

#### 在线生成 Banner 工具

可以使用在线工具生成 ASCII 艺术字：

1. **ASCII Art Generator**: http://patorjk.com/software/taag/
2. **Text to ASCII Art**: https://www.ascii-art.de/
3. **Spring Boot Banner Generator**: https://www.bootschool.net/ascii

#### Banner 显示时机

Banner 在 Spring Boot 启动流程中的显示时机：

```
1. SpringApplication 创建
2. Banner 打印 ← 这里
3. ApplicationContext 创建
4. Bean 加载
5. ApplicationReadyEvent 发布
```

#### 性能考虑

- **关闭 Banner**：在生产环境可以关闭 Banner 以略微提升启动速度
- **文件大小**：Banner 文件过大可能影响启动速度（通常影响很小）
- **日志输出**：如果使用 `LOG` 模式，Banner 会写入日志文件

#### 最佳实践

1. **开发环境**：使用自定义 Banner，增加开发体验
2. **生产环境**：可以关闭 Banner 或使用简洁版本
3. **多环境配置**：不同环境使用不同的 Banner
4. **信息展示**：利用 Banner 显示关键信息（版本、环境等）

**示例：根据环境显示不同 Banner**
```yaml
# application-dev.yml
spring:
  banner:
    location: classpath:banner-dev.txt

# application-prod.yml
spring:
  banner:
    mode: off  # 生产环境关闭
```

#### 常见问题

**Q1: Banner 文件找不到？**
- 检查文件路径是否正确（`classpath:banner.txt`）
- 确认文件在 `src/main/resources/` 目录下
- 检查文件编码（建议使用 UTF-8）

**Q2: Banner 中的变量不生效？**
- 确保变量名正确（如 `${application.name}`）
- 检查配置文件中是否定义了对应属性
- 使用默认值语法：`${property:defaultValue}`

**Q3: 如何在不同环境显示不同 Banner？**
- 使用 Profile 配置不同的 `banner.location`
- 或在 Banner 文件中使用条件判断（通过变量）

**Q4: Banner 影响启动性能吗？**
- 影响很小，通常可以忽略
- 如果 Banner 文件很大（>10KB），可能有轻微影响
- 生产环境建议关闭或使用简洁版本

---

## Web 开发

1. **@Controller vs @RestController**
   - 要点：`@Controller` 用于传统 MVC，返回视图；`@RestController` 用于 REST API，返回 JSON/XML。
   - 追问：`@RestController` 是 `@Controller` + `@ResponseBody` 的组合；如何混用。

### @Controller 和 @RestController 详解

#### 核心区别

**@Controller**
- **用途**：用于传统的 Spring MVC 应用，处理 Web 请求
- **返回值**：通常返回视图名称（String），由视图解析器解析为 HTML 页面
- **组合注解**：需要配合 `@ResponseBody` 才能返回 JSON/XML 数据
- **适用场景**：需要返回 HTML 页面的传统 Web 应用

**@RestController**
- **用途**：专门用于 RESTful API，处理 JSON/XML 数据
- **返回值**：直接返回对象，自动序列化为 JSON/XML（通过 `@ResponseBody`）
- **组合注解**：`@Controller` + `@ResponseBody` 的组合
- **适用场景**：前后端分离的 API 服务、微服务接口

#### 源码分析

```java
// @RestController 的定义
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody  // 关键：自动添加了 @ResponseBody
public @interface RestController {
    @AliasFor(annotation = Controller.class)
    String value() default "";
}
```

从源码可以看出，`@RestController` 实际上就是 `@Controller` + `@ResponseBody` 的组合。

#### 使用示例对比

**@Controller 示例（返回视图）**
```java
@Controller
@RequestMapping("/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    // 返回视图名称，会解析为 /templates/user/list.html
    @GetMapping("/list")
    public String listUsers(Model model) {
        List<User> users = userService.findAll();
        model.addAttribute("users", users);
        return "user/list";  // 视图名称
    }
    
    // 如果需要返回 JSON，需要添加 @ResponseBody
    @GetMapping("/api/list")
    @ResponseBody
    public List<User> listUsersApi() {
        return userService.findAll();  // 返回 JSON
    }
}
```

**@RestController 示例（返回 JSON）**
```java
@RestController
@RequestMapping("/api/users")
public class UserRestController {
    
    @Autowired
    private UserService userService;
    
    // 直接返回对象，自动序列化为 JSON
    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();  // 自动转为 JSON
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(user);  // 自动转为 JSON
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User saved = userService.save(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }
}
```

#### 混用场景

在同一个应用中，可以同时使用 `@Controller` 和 `@RestController`：

```java
// 传统 MVC 控制器（返回 HTML）
@Controller
@RequestMapping("/web")
public class WebController {
    
    @GetMapping("/users")
    public String userPage() {
        return "users";  // 返回视图
    }
}

// REST API 控制器（返回 JSON）
@RestController
@RequestMapping("/api")
public class ApiController {
    
    @GetMapping("/users")
    public List<User> getUsers() {
        return userService.findAll();  // 返回 JSON
    }
}
```

#### 方法级别的 @ResponseBody

即使使用 `@Controller`，也可以在方法上使用 `@ResponseBody` 返回 JSON：

```java
@Controller
@RequestMapping("/users")
public class UserController {
    
    // 返回视图
    @GetMapping("/page")
    public String userPage() {
        return "user/list";
    }
    
    // 返回 JSON（方法级别 @ResponseBody）
    @GetMapping("/api")
    @ResponseBody
    public List<User> getUsers() {
        return userService.findAll();
    }
}
```

#### 关键区别总结

| 特性 | @Controller | @RestController |
|------|-------------|-----------------|
| **组合注解** | 单独使用 | `@Controller` + `@ResponseBody` |
| **返回值类型** | 视图名称（String）或 ModelAndView | 对象（自动序列化） |
| **响应内容** | HTML 页面 | JSON/XML 数据 |
| **视图解析** | 需要视图解析器 | 不需要视图解析器 |
| **Content-Type** | `text/html` | `application/json` |
| **适用场景** | 传统 Web 应用 | RESTful API、前后端分离 |
| **方法级 @ResponseBody** | 需要手动添加 | 自动包含 |
| **性能（响应时间）** | 较慢（10-50ms） | 较快（1-5ms） |
| **性能（CPU 占用）** | 较高（模板渲染） | 较低（序列化） |
| **性能（内存占用）** | 较高（模板缓存） | 较低 |

#### 常见面试追问

**Q1: 为什么需要 @RestController？**
- 简化 REST API 开发，避免每个方法都写 `@ResponseBody`
- 语义更清晰，明确表示这是 REST 控制器
- 减少代码冗余，提高开发效率

**Q2: @Controller + @ResponseBody 和 @RestController 有什么区别？**
- 功能上完全等价
- `@RestController` 是语法糖，代码更简洁
- 语义上 `@RestController` 更明确表达 REST API 的意图

**Q3: 可以在 @RestController 中返回视图吗？**
- 可以，但不推荐
- 需要返回 `ModelAndView` 或使用 `@ResponseBody(false)`（Spring 5.3+）
- 建议：REST API 用 `@RestController`，视图用 `@Controller`

**Q4: 如何统一设置响应格式？**
```java
@RestController
@RequestMapping("/api")
public class ApiController {
    
    // 使用 ResponseEntity 控制响应
    @GetMapping("/users")
    public ResponseEntity<ApiResponse<List<User>>> getUsers() {
        List<User> users = userService.findAll();
        return ResponseEntity.ok(ApiResponse.success(users));
    }
}

// 或者使用 @ControllerAdvice 统一处理
@RestControllerAdvice
public class ResponseWrapper implements ResponseBodyAdvice<Object> {
    @Override
    public Object beforeBodyWrite(Object body, ...) {
        return ApiResponse.success(body);
    }
}
```

**Q5: @Controller 和 @RestController 哪个性能更高？**

这是一个常见的性能问题，答案取决于具体场景：

#### 性能对比分析

**1. 处理流程差异**

**@Controller（返回视图）：**
```
请求 → 控制器方法 → 视图解析器查找模板 → 模板引擎渲染 → HTML 响应
      ↓
   额外开销：视图解析、模板渲染、HTML 生成
```

**@RestController（返回 JSON）：**
```
请求 → 控制器方法 → Jackson 序列化 → JSON 响应
      ↓
   开销：仅序列化
```

**2. 性能对比**

| 指标 | @Controller（视图） | @RestController（JSON） | 说明 |
|------|-------------------|----------------------|------|
| **响应时间** | 较慢（10-50ms） | 较快（1-5ms） | 视图渲染需要时间 |
| **CPU 占用** | 较高 | 较低 | 模板引擎需要 CPU |
| **内存占用** | 较高 | 较低 | 模板缓存占用内存 |
| **网络传输** | 较大（HTML） | 较小（JSON） | JSON 更紧凑 |
| **可缓存性** | 中等 | 高 | JSON 更适合缓存 |

**3. 实际性能测试**

**测试场景：返回用户列表（100 条数据）**

```java
// @Controller 方式
@Controller
public class UserController {
    @GetMapping("/users")
    public String getUsers(Model model) {
        List<User> users = userService.findAll(); // 100 条
        model.addAttribute("users", users);
        return "users/list"; // Thymeleaf 模板
    }
}

// @RestController 方式
@RestController
public class UserRestController {
    @GetMapping("/api/users")
    public List<User> getUsers() {
        return userService.findAll(); // 100 条
    }
}
```

**性能测试结果（参考值）：**

| 场景 | @Controller | @RestController | 差异 |
|------|-------------|-----------------|------|
| **简单数据（10条）** | ~15ms | ~2ms | **7.5倍** |
| **中等数据（100条）** | ~45ms | ~5ms | **9倍** |
| **复杂数据（1000条）** | ~200ms | ~15ms | **13倍** |
| **高并发（1000 QPS）** | CPU 80% | CPU 30% | **2.7倍** |

**4. 性能差异原因**

**@Controller 的额外开销：**

1. **视图解析器查找**（~2-5ms）
   ```java
   // 需要查找模板文件
   ViewResolver.resolveViewName("users/list", locale)
   ```

2. **模板引擎渲染**（~10-40ms）
   ```java
   // Thymeleaf/Freemarker 需要解析模板、处理变量、生成 HTML
   templateEngine.process("users/list", context)
   ```

3. **HTML 生成**（~5-10ms）
   ```java
   // 生成完整的 HTML 文档（DOCTYPE、head、body 等）
   ```

4. **模板缓存**（内存占用）
   ```java
   // 模板引擎需要缓存编译后的模板
   ```

**@RestController 的优势：**

1. **直接序列化**（~1-5ms）
   ```java
   // Jackson 直接序列化对象为 JSON，无需额外处理
   objectMapper.writeValueAsString(user)
   ```

2. **无视图解析**（节省时间）
   ```java
   // 跳过视图解析器查找
   ```

3. **高效序列化**（Jackson 优化）
   ```java
   // Jackson 使用高效的序列化算法
   ```

#### 视图解析和模板渲染详解

为了更好地理解性能差异，我们需要深入理解**视图解析**和**模板渲染**这两个概念。

##### 什么是视图解析（View Resolution）？

**视图解析**是 Spring MVC 将控制器返回的视图名称（如 `"users/list"`）转换为实际视图对象（View）的过程。

**工作流程：**

```
控制器返回 "users/list"
    ↓
ViewResolver 查找模板文件
    ↓
找到 /templates/users/list.html
    ↓
创建 View 对象
    ↓
准备渲染
```

**代码示例：**

```java
@Controller
public class UserController {
    
    @GetMapping("/users")
    public String listUsers(Model model) {
        List<User> users = userService.findAll();
        model.addAttribute("users", users);
        return "users/list";  // ← 这是视图名称，不是文件路径
    }
}
```

**视图解析器的工作：**

```java
// Spring MVC 内部流程（简化版）
public class DispatcherServlet {
    
    public void processHandlerResult(...) {
        // 1. 获取视图名称
        String viewName = "users/list";
        
        // 2. 视图解析器查找
        View view = viewResolver.resolveViewName(viewName, locale);
        // viewResolver 会：
        // - 查找 /templates/users/list.html
        // - 检查文件是否存在
        // - 创建 ThymeleafView 对象
        
        // 3. 渲染视图
        view.render(model, request, response);
    }
}
```

**视图解析器类型：**

```java
// 1. Thymeleaf 视图解析器
ThymeleafViewResolver resolver = new ThymeleafViewResolver();
resolver.setPrefix("classpath:/templates/");  // 模板前缀
resolver.setSuffix(".html");                  // 模板后缀
// "users/list" → "/templates/users/list.html"

// 2. JSP 视图解析器
InternalResourceViewResolver resolver = new InternalResourceViewResolver();
resolver.setPrefix("/WEB-INF/views/");
resolver.setSuffix(".jsp");
// "users/list" → "/WEB-INF/views/users/list.jsp"

// 3. Freemarker 视图解析器
FreeMarkerViewResolver resolver = new FreeMarkerViewResolver();
resolver.setPrefix("");
resolver.setSuffix(".ftl");
// "users/list" → "users/list.ftl"
```

**视图解析的步骤：**

1. **接收视图名称**：`"users/list"`
2. **添加前缀和后缀**：`/templates/` + `users/list` + `.html`
3. **查找文件**：检查 `/templates/users/list.html` 是否存在
4. **创建 View 对象**：根据模板类型创建对应的 View
5. **返回 View**：供后续渲染使用

**性能开销：**
- 文件系统查找：~1-2ms
- 创建 View 对象：~0.5-1ms
- 模板缓存查找：~0.5-1ms
- **总计：~2-5ms**

##### 什么是模板渲染（Template Rendering）？

**模板渲染**是将模板文件（HTML 模板）和数据模型（Model）结合，生成最终 HTML 的过程。

**工作流程：**

```
模板文件（users/list.html）
    +
数据模型（users 列表）
    ↓
模板引擎处理
    ↓
生成 HTML 字符串
    ↓
写入 HTTP 响应
```

**模板文件示例（Thymeleaf）：**

```html
<!-- /templates/users/list.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>用户列表</title>
</head>
<body>
    <h1>用户列表</h1>
    <table>
        <tr th:each="user : ${users}">
            <td th:text="${user.id}">1</td>
            <td th:text="${user.name}">John</td>
            <td th:text="${user.email}">john@example.com</td>
        </tr>
    </table>
</body>
</html>
```

**模板渲染的详细步骤：**

**步骤 1：加载模板文件**
```java
// Thymeleaf 内部流程
Template template = templateEngine.getTemplate("users/list");
// 从文件系统或缓存中加载模板
// 如果未缓存，需要解析模板语法
```

**步骤 2：解析模板语法**
```java
// 解析 Thymeleaf 指令
// th:each="user : ${users}"  → 循环指令
// th:text="${user.name}"     → 文本替换指令
// ${users}                   → 变量表达式
```

**步骤 3：处理变量表达式**
```java
// 从 Model 中获取数据
List<User> users = (List<User>) model.get("users");
// users = [User(id=1, name="John"), User(id=2, name="Jane")]
```

**步骤 4：执行模板指令**
```java
// 处理 th:each 循环
for (User user : users) {
    // 处理每一行
    // th:text="${user.name}" → 替换为 "John"
}
```

**步骤 5：生成 HTML**
```java
// 生成最终的 HTML
String html = """
    <!DOCTYPE html>
    <html>
    <head><title>用户列表</title></head>
    <body>
        <h1>用户列表</h1>
        <table>
            <tr><td>1</td><td>John</td><td>john@example.com</td></tr>
            <tr><td>2</td><td>Jane</td><td>jane@example.com</td></tr>
        </table>
    </body>
    </html>
    """;
```

**步骤 6：写入响应**
```java
response.getWriter().write(html);
response.setContentType("text/html;charset=UTF-8");
```

**完整的渲染过程（代码示例）：**

```java
// Thymeleaf 模板引擎内部（简化版）
public class ThymeleafView {
    
    public void render(Map<String, Object> model, 
                      HttpServletRequest request,
                      HttpServletResponse response) {
        
        // 1. 获取模板（从缓存或文件系统）
        Template template = getTemplate("users/list");
        
        // 2. 创建上下文
        Context context = new Context();
        context.setVariables(model);  // 设置 users 变量
        
        // 3. 处理模板
        String html = templateEngine.process("users/list", context);
        // 这一步包括：
        // - 解析模板语法
        // - 处理变量表达式 ${users}
        // - 执行指令 th:each, th:text
        // - 生成 HTML
        
        // 4. 写入响应
        response.setContentType("text/html;charset=UTF-8");
        response.getWriter().write(html);
    }
}
```

**模板渲染的性能开销：**

| 步骤 | 操作 | 耗时 | 说明 |
|------|------|------|------|
| **1. 加载模板** | 文件读取/缓存查找 | ~1-2ms | 首次需要读取文件 |
| **2. 解析语法** | 解析 Thymeleaf 指令 | ~2-5ms | 解析模板语法树 |
| **3. 处理变量** | 从 Model 获取数据 | ~0.5ms | 数据访问 |
| **4. 执行指令** | 循环、条件判断 | ~5-20ms | 取决于数据量 |
| **5. 生成 HTML** | 字符串拼接 | ~2-10ms | 生成 HTML 字符串 |
| **6. 写入响应** | 网络传输 | ~1-3ms | 写入 HTTP 响应 |
| **总计** | | **~10-40ms** | 取决于模板复杂度 |

##### 对比：JSON 序列化（@RestController）

**@RestController 的处理流程：**

```java
@RestController
public class UserRestController {
    
    @GetMapping("/api/users")
    public List<User> getUsers() {
        return userService.findAll();
    }
}
```

**处理步骤：**

**步骤 1：获取数据**
```java
List<User> users = userService.findAll();
// users = [User(id=1, name="John"), User(id=2, name="Jane")]
```

**步骤 2：序列化为 JSON**
```java
// Jackson 自动序列化
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(users);
// json = "[{\"id\":1,\"name\":\"John\"},{\"id\":2,\"name\":\"Jane\"}]"
```

**步骤 3：写入响应**
```java
response.setContentType("application/json;charset=UTF-8");
response.getWriter().write(json);
```

**JSON 序列化的性能开销：**

| 步骤 | 操作 | 耗时 | 说明 |
|------|------|------|------|
| **1. 获取数据** | 数据访问 | ~0.5ms | 同 @Controller |
| **2. 序列化** | Jackson 序列化 | ~1-4ms | 高效的序列化算法 |
| **3. 写入响应** | 网络传输 | ~0.5-1ms | JSON 体积小 |
| **总计** | | **~1-5ms** | 比模板渲染快 5-10 倍 |

##### 完整对比示例

**场景：返回 100 个用户的数据**

**@Controller 方式：**

```java
// 1. 控制器方法（~1ms）
@GetMapping("/users")
public String listUsers(Model model) {
    List<User> users = userService.findAll();  // 100 条
    model.addAttribute("users", users);
    return "users/list";
}

// 2. 视图解析（~3ms）
// ViewResolver 查找 /templates/users/list.html

// 3. 模板渲染（~30ms）
// - 加载模板：~2ms
// - 解析语法：~5ms
// - 处理 100 条数据循环：~20ms
// - 生成 HTML：~3ms

// 4. 响应（~2ms）
// 写入 HTML（体积大，~50KB）

// 总计：~36ms
```

**@RestController 方式：**

```java
// 1. 控制器方法（~1ms）
@GetMapping("/api/users")
public List<User> getUsers() {
    return userService.findAll();  // 100 条
}

// 2. JSON 序列化（~3ms）
// Jackson 序列化 100 个对象

// 3. 响应（~1ms）
// 写入 JSON（体积小，~15KB）

// 总计：~5ms
```

**性能差异：36ms vs 5ms = 7.2 倍**

##### 为什么模板渲染更慢？

1. **额外的文件操作**
   - 需要读取模板文件（首次）
   - 需要解析模板语法

2. **复杂的处理逻辑**
   - 需要处理循环、条件判断
   - 需要处理各种模板指令

3. **更大的输出体积**
   - HTML 包含完整的文档结构
   - JSON 只包含数据

4. **更多的 CPU 计算**
   - 字符串拼接和格式化
   - 模板语法解析

##### 总结

| 概念 | 说明 | 耗时 | 是否必需 |
|------|------|------|---------|
| **视图解析** | 将视图名称转换为 View 对象 | ~2-5ms | @Controller 必需 |
| **模板渲染** | 将模板和数据生成 HTML | ~10-40ms | @Controller 必需 |
| **JSON 序列化** | 将对象转换为 JSON | ~1-5ms | @RestController 必需 |

**关键理解：**
- **视图解析**：找到模板文件的过程
- **模板渲染**：用数据填充模板生成 HTML 的过程
- **JSON 序列化**：直接将对象转为 JSON，无需模板

这就是为什么 @RestController 性能更高的根本原因！

**5. 实际场景分析**

**场景 1：简单页面（静态内容为主）**
- **@Controller**：性能可接受，用户体验好（服务端渲染）
- **@RestController**：不适用（需要前端渲染）

**场景 2：数据 API（纯数据）**
- **@Controller**：性能差，不适用
- **@RestController**：性能最优，推荐使用

**场景 3：混合场景（页面 + API）**
- **@Controller**：用于页面
- **@RestController**：用于 API
- 性能：各司其职，最优

**6. 性能优化建议**

**@Controller 性能优化：**

```java
// 1. 使用模板缓存
spring:
  thymeleaf:
    cache: true  # 生产环境启用缓存

// 2. 减少模板复杂度
// 避免在模板中执行复杂逻辑

// 3. 使用片段缓存
<div th:fragment="user-list" th:cacheable="true">
    <!-- 内容 -->
</div>
```

**@RestController 性能优化：**

```java
// 1. 使用 @JsonView 减少序列化字段
@JsonView(Views.Public.class)
public class User {
    // ...
}

// 2. 使用流式序列化（大数据量）
@GetMapping(value = "/users", produces = MediaType.APPLICATION_NDJSON_VALUE)
public Stream<User> getUsersStream() {
    return userService.findAllStream();
}

// 3. 启用压缩
server:
  compression:
    enabled: true
    mime-types: application/json
```

**7. 结论**

**性能排序（从快到慢）：**

1. **@RestController（JSON）** - 最快 ⭐⭐⭐⭐⭐
   - 适合：RESTful API、前后端分离
   - 性能：响应时间 1-5ms，CPU 占用低

2. **@Controller + @ResponseBody（JSON）** - 快 ⭐⭐⭐⭐
   - 功能等同于 @RestController
   - 性能：与 @RestController 相同

3. **@Controller（视图）** - 较慢 ⭐⭐⭐
   - 适合：传统 Web 应用、服务端渲染
   - 性能：响应时间 10-50ms，CPU 占用较高

**选择建议：**

- **纯 API 服务**：使用 `@RestController`，性能最优
- **传统 Web 应用**：使用 `@Controller`，性能可接受
- **混合应用**：按功能区分，API 用 `@RestController`，页面用 `@Controller`
- **高并发场景**：优先使用 `@RestController`，减少服务器压力

**性能测试代码示例：**

```java
@RestController
public class PerformanceTestController {
    
    @GetMapping("/test/controller")
    public String testController() {
        long start = System.currentTimeMillis();
        // 模拟视图渲染
        String html = renderView();
        long end = System.currentTimeMillis();
        return "Controller time: " + (end - start) + "ms";
    }
    
    @GetMapping("/test/restcontroller")
    public Map<String, Object> testRestController() {
        long start = System.currentTimeMillis();
        Map<String, Object> data = getData();
        long end = System.currentTimeMillis();
        data.put("processingTime", (end - start) + "ms");
        return data;
    }
}
```

#### 最佳实践

1. **前后端分离项目**：统一使用 `@RestController`
2. **传统 Web 项目**：使用 `@Controller` 返回视图
3. **混合项目**：按模块区分，API 模块用 `@RestController`，页面模块用 `@Controller`
4. **避免混用**：同一个类中不要同时返回视图和 JSON（除非有明确需求）

2. **Spring MVC 自动配置**
   - 要点：`WebMvcAutoConfiguration`；`DispatcherServlet` 自动注册；视图解析器。
   - 追问：如何自定义 MVC 配置；拦截器配置。

2. **静态资源处理**
   - 要点：默认路径（`/static`、`/public`、`/resources`、`/META-INF/resources`）；资源映射。
   - 追问：如何修改静态资源路径；CDN 集成。

3. **JSON 序列化与内容协商**
   - 要点：Jackson 自动配置；`@JsonFormat`、`@JsonIgnore`；内容协商机制；XML 支持。
   - 追问：如何切换 JSON 库；日期格式处理；为什么 @RestController 可以返回 XML。

### 内容协商（Content Negotiation）详解

#### 为什么 @RestController 可以返回 XML？

`@RestController` 不仅仅返回 JSON，它可以根据客户端的请求自动返回 JSON 或 XML，这得益于 Spring MVC 的**内容协商（Content Negotiation）**机制。

#### 内容协商原理

**内容协商**是 HTTP 协议的一个特性，允许客户端和服务器协商响应的内容格式。Spring MVC 通过以下方式确定返回格式：

1. **Accept 请求头**：客户端通过 `Accept` 头声明期望的媒体类型
2. **URL 扩展名**：通过 URL 后缀（如 `.json`、`.xml`）指定格式
3. **查询参数**：通过 `format` 参数指定格式

#### 工作流程

```
客户端请求
    ↓
检查 Accept 头 / URL 扩展名 / 查询参数
    ↓
确定媒体类型（MediaType）
    ↓
查找对应的 HttpMessageConverter
    ↓
序列化对象为对应格式
    ↓
返回响应（Content-Type 头）
```

#### 默认支持的格式

Spring Boot 默认支持以下格式：

| 格式 | MediaType | 依赖 | 说明 |
|------|-----------|------|------|
| **JSON** | `application/json` | Jackson（默认包含） | 默认格式 |
| **XML** | `application/xml` | Jackson XML（需添加） | 需要额外依赖 |
| **文本** | `text/plain` | 内置 | 简单文本 |

#### 如何启用 XML 支持

**步骤 1：添加依赖**

```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

**步骤 2：配置内容协商（可选）**

```yaml
# application.yml
spring:
  mvc:
    contentnegotiation:
      favor-parameter: true      # 支持 format 参数
      favor-path-extension: true # 支持 URL 扩展名
      parameter-name: format     # 参数名称
      media-types:
        json: application/json
        xml: application/xml
```

#### 实际示例

**示例 1：通过 Accept 头控制返回格式**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

**请求示例：**

```bash
# 返回 JSON（默认）
curl -H "Accept: application/json" http://localhost:8080/api/users/1
# 响应：Content-Type: application/json
# {"id":1,"name":"John","email":"john@example.com"}

# 返回 XML
curl -H "Accept: application/xml" http://localhost:8080/api/users/1
# 响应：Content-Type: application/xml
# <?xml version="1.0" encoding="UTF-8"?>
# <User>
#   <id>1</id>
#   <name>John</name>
#   <email>john@example.com</email>
# </User>
```

**示例 2：通过 URL 扩展名控制**

```yaml
# 启用 URL 扩展名支持
spring:
  mvc:
    contentnegotiation:
      favor-path-extension: true
```

```bash
# JSON 格式
curl http://localhost:8080/api/users/1.json

# XML 格式
curl http://localhost:8080/api/users/1.xml
```

**示例 3：通过查询参数控制**

```yaml
# 启用查询参数支持
spring:
  mvc:
    contentnegotiation:
      favor-parameter: true
      parameter-name: format
```

```bash
# JSON 格式
curl http://localhost:8080/api/users/1?format=json

# XML 格式
curl http://localhost:8080/api/users/1?format=xml
```

#### HttpMessageConverter 机制

Spring MVC 使用 `HttpMessageConverter` 来序列化/反序列化对象：

```java
// Spring Boot 自动配置的 Converter
1. MappingJackson2HttpMessageConverter  // JSON（默认）
2. MappingJackson2XmlHttpMessageConverter // XML（需要依赖）
3. StringHttpMessageConverter            // 文本
4. ByteArrayHttpMessageConverter         // 字节数组
```

**查看已注册的 Converter：**

```java
@RestController
public class ConverterController {
    
    @Autowired
    private RequestMappingHandlerAdapter adapter;
    
    @GetMapping("/converters")
    public List<String> getConverters() {
        return adapter.getMessageConverters().stream()
                .map(c -> c.getClass().getSimpleName())
                .collect(Collectors.toList());
    }
}
```

#### 强制指定返回格式

**方式 1：使用 produces 属性**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    // 强制返回 JSON
    @GetMapping(value = "/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
    public User getUserAsJson(@PathVariable Long id) {
        return userService.findById(id);
    }
    
    // 强制返回 XML
    @GetMapping(value = "/{id}/xml", produces = MediaType.APPLICATION_XML_VALUE)
    public User getUserAsXml(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

**方式 2：使用 ResponseEntity**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(
            @PathVariable Long id,
            @RequestHeader(value = "Accept", defaultValue = "application/json") String accept) {
        
        User user = userService.findById(id);
        HttpHeaders headers = new HttpHeaders();
        
        if (accept.contains("xml")) {
            headers.setContentType(MediaType.APPLICATION_XML);
        } else {
            headers.setContentType(MediaType.APPLICATION_JSON);
        }
        
        return ResponseEntity.ok().headers(headers).body(user);
    }
}
```

#### XML 序列化配置

**Jackson XML 注解：**

```java
@JacksonXmlRootElement(localName = "user")
public class User {
    
    @JacksonXmlProperty(localName = "user_id")
    private Long id;
    
    @JacksonXmlProperty(localName = "user_name")
    private String name;
    
    @JacksonXmlElementWrapper(localName = "addresses")
    @JacksonXmlProperty(localName = "address")
    private List<Address> addresses;
    
    @JacksonXmlText  // 文本内容
    private String description;
    
    // getters and setters
}
```

**XML 输出示例：**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<user>
    <user_id>1</user_id>
    <user_name>John</user_name>
    <addresses>
        <address>
            <street>123 Main St</street>
            <city>New York</city>
        </address>
    </addresses>
    <description>User description</description>
</user>
```

#### 内容协商优先级

Spring MVC 按以下顺序确定媒体类型：

1. **URL 扩展名**（如果启用 `favor-path-extension`）
2. **查询参数**（如果启用 `favor-parameter`）
3. **Accept 请求头**
4. **默认格式**（通常是 JSON）

#### 配置示例

**完整的内容协商配置：**

```yaml
spring:
  mvc:
    contentnegotiation:
      # 支持查询参数
      favor-parameter: true
      parameter-name: format
      
      # 支持 URL 扩展名
      favor-path-extension: true
      
      # 默认媒体类型
      default-content-type: application/json
      
      # 支持的媒体类型映射
      media-types:
        json: application/json
        xml: application/xml
        yaml: application/x-yaml
```

#### 常见问题

**Q1: 为什么默认返回 JSON？**

- Spring Boot 默认包含 Jackson JSON 依赖
- JSON 是 RESTful API 的主流格式
- 性能好、可读性强、体积小

**Q2: 如何禁用 XML 支持？**

```yaml
spring:
  mvc:
    contentnegotiation:
      media-types:
        xml: null  # 移除 XML 支持
```

**Q3: 如何同时支持 JSON 和 XML？**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    // 同时支持 JSON 和 XML
    @GetMapping(value = "/{id}", 
                produces = {MediaType.APPLICATION_JSON_VALUE, 
                           MediaType.APPLICATION_XML_VALUE})
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

**Q4: 如何自定义序列化器？**

```java
@Configuration
public class JacksonConfig {
    
    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jsonCustomizer() {
        return builder -> {
            builder.serializationInclusion(JsonInclude.Include.NON_NULL);
            builder.simpleDateFormat("yyyy-MM-dd HH:mm:ss");
        };
    }
    
    @Bean
    public Jackson2ObjectMapperBuilderCustomizer xmlCustomizer() {
        return builder -> {
            builder.xml().defaultUseWrapper(false);
        };
    }
}
```

#### 最佳实践

1. **默认使用 JSON**：JSON 是 RESTful API 的标准格式
2. **按需支持 XML**：只在需要时添加 XML 支持
3. **明确指定格式**：使用 `produces` 明确支持的格式
4. **统一格式**：同一 API 尽量统一返回格式
5. **性能考虑**：XML 序列化比 JSON 慢，注意性能影响

3. **JSON 序列化**
   - 要点：Jackson 自动配置；`@JsonFormat`、`@JsonIgnore`；自定义序列化器。
   - 追问：如何切换 JSON 库；日期格式处理。

4. **异常处理**
   - 要点：`@ControllerAdvice`、`@ExceptionHandler`；全局异常处理。
   - 追问：异常处理优先级；RESTful API 异常响应。

5. **跨域配置（CORS）**
   - 要点：`@CrossOrigin`；`WebMvcConfigurer.addCorsMappings()`；全局配置。
   - 追问：CORS 预检请求；安全考虑。

6. **文件上传**
   - 要点：`MultipartFile`；`spring.servlet.multipart` 配置；大文件上传。
   - 追问：文件大小限制；临时文件清理。

### 示例：全局异常处理
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleIllegalArgument(IllegalArgumentException e) {
        ErrorResponse error = new ErrorResponse("BAD_REQUEST", e.getMessage());
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        ErrorResponse error = new ErrorResponse("INTERNAL_ERROR", "An error occurred");
        return ResponseEntity.status(500).body(error);
    }
}
```

### 示例：CORS 配置
```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:3000")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

---

## 数据访问

1. **数据源自动配置**
   - 要点：`DataSourceAutoConfiguration`；HikariCP 默认连接池；多数据源配置。
   - 追问：如何切换连接池；连接池参数调优。

2. **JPA/Hibernate 自动配置**
   - 要点：`JpaRepositoriesAutoConfiguration`；实体扫描；事务管理。
   - 追问：如何自定义 JPA 配置；N+1 查询问题。

3. **MyBatis 集成**
   - 要点：`mybatis-spring-boot-starter`；Mapper 扫描；配置属性。
   - 追问：MyBatis 与 JPA 选择；分页插件。

4. **Redis 自动配置**
   - 要点：`RedisAutoConfiguration`；`RedisTemplate`、`StringRedisTemplate`；连接池配置。
   - 追问：Redis 序列化；集群配置。

5. **MongoDB 自动配置**
   - 要点：`MongoAutoConfiguration`；`MongoTemplate`；Repository 支持。
   - 追问：MongoDB 事务；索引管理。

### 示例：多数据源配置
```java
@Configuration
public class DataSourceConfig {
    
    @Primary
    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public JdbcTemplate primaryJdbcTemplate(@Qualifier("primaryDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
    
    @Bean
    public JdbcTemplate secondaryJdbcTemplate(@Qualifier("secondaryDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

### 示例：Redis 配置
```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: 
    database: 0
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0
        max-wait: -1ms
```

---

## 监控与管理（Actuator）

1. **Actuator 端点**
   - 要点：`/actuator/health`、`/actuator/info`、`/actuator/metrics`、`/actuator/env`。
   - 追问：如何暴露端点；端点安全配置。

2. **健康检查**
   - 要点：`HealthIndicator`；自定义健康检查；聚合健康状态。
   - 追问：数据库健康检查；外部服务健康检查。

3. **指标收集**
   - 要点：Micrometer；Prometheus 集成；自定义指标。
   - 追问：如何导出指标；指标聚合。

4. **信息端点**
   - 要点：`management.info.*`；Git 信息；构建信息。
   - 追问：如何自定义信息；版本信息展示。

5. **日志管理**
   - 要点：`/actuator/loggers`；动态修改日志级别。
   - 追问：生产环境使用；安全考虑。

### 示例：自定义健康检查
```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        // 检查逻辑
        if (checkService()) {
            return Health.up()
                    .withDetail("service", "available")
                    .build();
        } else {
            return Health.down()
                    .withDetail("service", "unavailable")
                    .withException(new RuntimeException("Service check failed"))
                    .build();
        }
    }
    
    private boolean checkService() {
        // 实际检查逻辑
        return true;
    }
}
```

### 示例：Actuator 配置
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
  metrics:
    export:
      prometheus:
        enabled: true
```

---

## 测试

1. **测试自动配置**
   - 要点：`@SpringBootTest`；`@MockBean`；测试切片（`@WebMvcTest`、`@DataJpaTest`）。
   - 追问：如何提高测试速度；测试隔离。

2. **Web 层测试**
   - 要点：`MockMvc`；`@AutoConfigureMockMvc`；请求/响应验证。
   - 追问：如何测试异常场景；异步请求测试。

3. **数据层测试**
   - 要点：`@DataJpaTest`；嵌入式数据库；测试数据准备。
   - 追问：如何避免测试污染；事务回滚。

4. **集成测试**
   - 要点：`@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)`；`TestRestTemplate`。
   - 追问：如何测试外部服务；Mock 外部依赖。

5. **测试配置**
   - 要点：`@TestConfiguration`；测试专用配置；Profile 切换。
   - 追问：如何复用测试配置；测试环境隔离。

### 示例：Web 层测试
```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void testGetUser() throws Exception {
        when(userService.findById(1L)).thenReturn(new User(1L, "John"));
        
        mockMvc.perform(get("/api/users/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("John"));
    }
}
```

### 示例：数据层测试
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @Transactional
    @Rollback
    void testSaveUser() {
        User user = new User("John");
        User saved = userRepository.save(user);
        assertThat(saved.getId()).isNotNull();
    }
}
```

---

## 部署与打包

1. **打包方式**
   - 要点：JAR（可执行 JAR）、WAR（传统部署）；Fat JAR 结构。
   - 追问：如何选择打包方式；Docker 部署。

### JAR 类型详解

#### 什么是 Fat JAR（胖 JAR）

**Fat JAR**（也称为 **Uber JAR** 或 **JAR with dependencies**）是一种包含所有依赖的 JAR 文件。它将应用程序代码、所有第三方依赖库、以及资源文件打包到一个 JAR 文件中，形成一个"胖"的、自包含的可执行文件。

**特点：**
- ✅ 包含所有依赖，无需外部类路径
- ✅ 单个文件，部署简单
- ✅ 可以直接运行：`java -jar app.jar`
- ❌ 文件体积大（通常 50-200MB）
- ❌ 依赖更新需要重新打包整个 JAR

#### JAR 类型对比

| JAR 类型 | 说明 | 包含内容 | 文件大小 | 适用场景 |
|---------|------|---------|---------|---------|
| **Fat JAR** | 包含所有依赖的 JAR | 应用代码 + 所有依赖 | 大（50-200MB） | Spring Boot 默认，简单部署 |
| **Thin JAR** | 不包含依赖的 JAR | 仅应用代码 | 小（<10MB） | 依赖外部管理，容器化部署 |
| **Uber JAR** | Fat JAR 的另一种叫法 | 同 Fat JAR | 同 Fat JAR | 同 Fat JAR |
| **Executable JAR** | 可执行的 JAR | 应用代码 + 依赖 + 启动类 | 大 | Spring Boot 应用 |
| **Library JAR** | 库 JAR | 仅库代码 | 小 | 作为依赖被其他项目使用 |

#### Spring Boot 的 Fat JAR 结构

Spring Boot 使用特殊的嵌套 JAR 结构（Nested JAR），这是 Fat JAR 的一种实现方式：

```
app.jar
├── META-INF/
│   ├── MANIFEST.MF          # 包含 Main-Class 和 Start-Class
│   └── maven/               # Maven 元数据
├── BOOT-INF/
│   ├── classes/             # 应用编译后的类文件
│   │   ├── com/example/Application.class
│   │   └── application.yml
│   └── lib/                 # 所有依赖 JAR（嵌套）
│       ├── spring-boot-2.7.0.jar
│       ├── spring-core-5.3.21.jar
│       ├── jackson-core-2.13.3.jar
│       └── ... (所有依赖)
└── org/springframework/boot/loader/
    ├── JarLauncher.class    # Spring Boot 的类加载器
    └── ...
```

**关键点：**
- `BOOT-INF/classes/`：存放应用代码
- `BOOT-INF/lib/`：存放所有依赖 JAR（嵌套结构）
- `org/springframework/boot/loader/`：Spring Boot 自定义类加载器

#### 查看 JAR 内容

```bash
# 查看 JAR 文件结构
jar -tf app.jar | head -20

# 查看 MANIFEST.MF
unzip -p app.jar META-INF/MANIFEST.MF

# 解压 JAR（查看完整结构）
unzip -q app.jar -d app-extracted
tree app-extracted
```

**MANIFEST.MF 示例：**
```
Manifest-Version: 1.0
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: com.example.Application
Spring-Boot-Version: 2.7.0
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
```

#### 不同类型的 JAR 实现

**1. Fat JAR（Spring Boot 默认）**

Maven 配置：
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>repackage</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

打包命令：
```bash
mvn clean package
# 生成：target/app-1.0.0.jar（Fat JAR，包含所有依赖）
```

**2. Thin JAR（瘦 JAR）**

Thin JAR 不包含依赖，依赖在运行时从 Maven 仓库下载。

Maven 配置：
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <layout>JAR</layout>
                <includes>
                    <include>
                        <groupId>nothing</groupId>
                        <artifactId>nothing</artifactId>
                    </include>
                </includes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

或使用 Spring Boot Thin Launcher：
```xml
<dependency>
    <groupId>org.springframework.boot.experimental</groupId>
    <artifactId>spring-boot-thin-layout</artifactId>
    <version>1.0.28.RELEASE</version>
</dependency>
```

**3. 普通 JAR（Library JAR）**

不包含依赖，仅应用代码：
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

**4. WAR 文件（传统 Web 应用）**

```xml
<packaging>war</packaging>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <packaging>war</packaging>
            </configuration>
        </plugin>
    </plugins>
</build>
```

#### 类加载机制

Spring Boot 使用自定义的 `JarLauncher` 来加载嵌套 JAR：

```java
// Spring Boot 的类加载流程
1. JarLauncher.main() 启动
2. 创建 LaunchedURLClassLoader
3. 从 BOOT-INF/lib/ 加载依赖 JAR
4. 从 BOOT-INF/classes/ 加载应用类
5. 调用 Start-Class（应用主类）
```

**为什么需要自定义类加载器？**
- 标准 Java 类加载器不支持嵌套 JAR
- 需要从 JAR 内的 JAR 加载类
- Spring Boot 实现了 `JarFile` 和 `JarURLConnection` 来支持嵌套结构

#### 性能对比

| 指标 | Fat JAR | Thin JAR | WAR |
|------|---------|----------|-----|
| **启动时间** | 中等 | 快（依赖已缓存） | 慢（需要解压） |
| **文件大小** | 大 | 小 | 中等 |
| **部署复杂度** | 简单 | 中等（需管理依赖） | 复杂（需容器） |
| **依赖管理** | 内置 | 外部 | 外部 |
| **容器化** | 适合 | 更适合 | 需要 |

#### 选择建议

**使用 Fat JAR 如果：**
- ✅ 需要简单部署（单个文件）
- ✅ 微服务架构
- ✅ 容器化部署（Docker）
- ✅ 不需要频繁更新依赖
- ✅ 网络环境不稳定

**使用 Thin JAR 如果：**
- ✅ 多个服务共享依赖
- ✅ 依赖更新频繁
- ✅ 需要减小镜像体积
- ✅ 有稳定的 Maven 仓库访问

**使用 WAR 如果：**
- ✅ 传统企业环境
- ✅ 已有 Tomcat/Jetty 服务器
- ✅ 需要多应用共享服务器资源
- ✅ 需要服务器级别的配置

#### 实际示例

**Fat JAR 打包和运行：**
```bash
# 打包
mvn clean package

# 查看文件大小
ls -lh target/app-1.0.0.jar
# 输出：app-1.0.0.jar  85M

# 运行
java -jar target/app-1.0.0.jar

# 查看 JAR 内容
jar -tf target/app-1.0.0.jar | grep BOOT-INF/lib | wc -l
# 输出：150（包含 150 个依赖 JAR）
```

**Thin JAR 打包和运行：**
```bash
# 打包（使用 Thin Launcher）
mvn clean package

# 查看文件大小
ls -lh target/app-1.0.0.jar
# 输出：app-1.0.0.jar  5M（仅应用代码）

# 运行（首次会下载依赖）
java -jar target/app-1.0.0.jar
# 依赖会缓存在 ~/.m2/repository/

# 查看依赖列表
java -jar target/app-1.0.0.jar --thin.list
```

#### 常见问题

**Q1: Fat JAR 文件太大怎么办？**
- 使用 Thin JAR
- 排除不必要的依赖
- 使用 Docker 多阶段构建，共享基础镜像
- 压缩 JAR（`spring-boot-maven-plugin` 支持）

**Q2: 如何查看 JAR 中包含哪些依赖？**
```bash
# 列出所有依赖
jar -tf app.jar | grep "BOOT-INF/lib"

# 或使用 Maven 命令
mvn dependency:tree
```

**Q3: 可以同时生成 Fat JAR 和 Thin JAR 吗？**
```xml
<build>
    <plugins>
        <!-- 生成 Fat JAR（默认） -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <id>repackage</id>
                    <goals><goal>repackage</goal></goals>
                </execution>
            </executions>
        </plugin>
        
        <!-- 生成 Thin JAR（分类器） -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <id>thin-repackage</id>
                    <goals><goal>repackage</goal></goals>
                    <configuration>
                        <classifier>thin</classifier>
                        <!-- Thin JAR 配置 -->
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

**Q4: Docker 镜像中如何使用 Thin JAR？**
```dockerfile
# 使用 Thin JAR + 依赖缓存层
FROM maven:3.8-openjdk-17 AS dependencies
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline

FROM openjdk:17-jre-slim
WORKDIR /app
COPY --from=dependencies /root/.m2 /root/.m2
COPY target/app-1.0.0.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

2. **可执行 JAR**
   - 要点：`spring-boot-maven-plugin`；嵌套 JAR；启动类配置。
   - 追问：JAR 文件结构；类加载机制。

3. **Docker 化**
   - 要点：多阶段构建；镜像优化；健康检查。
   - 追问：如何减小镜像体积；构建缓存。

4. **生产环境配置**
   - 要点：外部化配置；环境变量；配置中心。
   - 追问：敏感信息管理；配置热更新。

5. **启动脚本**
   - 要点：`start.sh`；JVM 参数；日志配置。
   - 追问：如何实现优雅关闭；进程管理。

### 示例：Dockerfile
```dockerfile
# 多阶段构建
FROM maven:3.8-openjdk-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

FROM openjdk:17-jre-slim
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 示例：启动脚本
```bash
#!/bin/bash
APP_NAME=myapp
JAR_FILE=app.jar
JVM_OPTS="-Xms512m -Xmx1024m -XX:+UseG1GC"

java $JVM_OPTS -jar $JAR_FILE \
    --spring.profiles.active=prod \
    --server.port=8080
```

---

## 性能优化

1. **启动性能优化**
   - 要点：懒加载（`@Lazy`）；排除不必要的自动配置；减少 Bean 扫描。
   - 追问：如何分析启动时间；启动性能监控。

2. **运行时性能优化**
   - 要点：连接池调优；缓存配置；异步处理。
   - 追问：如何定位性能瓶颈；JVM 调优。

3. **内存优化**
   - 要点：堆内存设置；Metaspace 配置；GC 选择。
   - 追问：如何减少内存占用；内存泄漏排查。

4. **数据库优化**
   - 要点：连接池参数；查询优化；批量操作。
   - 追问：如何监控数据库性能；慢查询分析。

5. **缓存优化**
   - 要点：Spring Cache；缓存策略；缓存预热。
   - 追问：如何提高缓存命中率；缓存穿透处理。

### 示例：启动性能分析
```java
@SpringBootApplication
public class Application {
    
    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        SpringApplication app = new SpringApplication(Application.class);
        app.setBannerMode(Banner.Mode.OFF);
        ConfigurableApplicationContext context = app.run(args);
        long end = System.currentTimeMillis();
        System.out.println("Startup time: " + (end - start) + "ms");
    }
}
```

---

## 故障排查

1. **启动失败**
   - 要点：查看启动日志；Bean 创建失败；配置错误。
   - 方案：检查依赖；验证配置；查看异常堆栈。

2. **端口冲突**
   - 要点：`server.port` 配置；`PORT` 环境变量；多实例部署。
   - 方案：修改端口；使用随机端口；检查进程占用。

3. **Bean 冲突**
   - 要点：多个同类型 Bean；自动配置冲突；`@Primary` 使用。
   - 方案：排除自动配置；使用 `@Qualifier`；调整 Bean 顺序。

4. **配置不生效**
   - 要点：配置优先级；Profile 未激活；属性绑定失败。
   - 方案：检查配置顺序；验证属性名；启用调试日志。

5. **内存溢出**
   - 要点：堆内存不足；Metaspace 溢出；直接内存溢出。
   - 方案：增加内存；分析堆转储；优化代码。

### 示例：启动调试
```yaml
# application.yml
logging:
  level:
    org.springframework.boot.autoconfigure: DEBUG
    org.springframework.context: DEBUG
    root: INFO
```

---

## 最佳实践

1. **项目结构**
   - 要点：包结构规范；分层架构；配置分离。
   - 追问：如何组织大型项目；模块化设计。

2. **配置管理**
   - 要点：环境隔离；敏感信息加密；配置版本管理。
   - 追问：如何管理多环境配置；配置中心选择。

3. **异常处理**
   - 要点：统一异常处理；错误码设计；日志记录。
   - 追问：如何设计异常体系；异常监控。

4. **日志管理**
   - 要点：日志级别；日志格式；日志聚合。
   - 追问：如何实现结构化日志；日志性能影响。

5. **安全实践**
   - 要点：依赖安全扫描；HTTPS；认证授权。
   - 追问：如何防范常见攻击；安全审计。

### 示例：统一异常处理
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ApiResponse<?>> handleBusinessException(BusinessException e) {
        logger.warn("Business exception: {}", e.getMessage());
        return ResponseEntity.ok(ApiResponse.error(e.getCode(), e.getMessage()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<?>> handleException(Exception e) {
        logger.error("Unexpected error", e);
        return ResponseEntity.status(500)
                .body(ApiResponse.error("INTERNAL_ERROR", "Internal server error"));
    }
}
```

---

## 常见场景题

1. **如何实现多环境配置？**
   - 要点：使用 Profile；外部配置文件；配置中心。
   - 方案：`application-{profile}.yml`；`spring.profiles.active`；环境变量。

2. **如何自定义 Starter？**
   - 要点：创建自动配置类；编写 `spring.factories`；提供默认配置。
   - 方案：遵循命名规范；提供条件注解；文档完善。

3. **如何实现配置热更新？**
   - 要点：`@RefreshScope`；配置中心；事件监听。
   - 方案：Spring Cloud Config；Nacos；Apollo。

4. **如何优化启动时间？**
   - 要点：排除不必要的自动配置；懒加载；减少 Bean 扫描。
   - 方案：分析启动日志；使用 `@Lazy`；优化依赖。

5. **如何实现优雅关闭？**
   - 要点：`spring.lifecycle.timeout-per-shutdown-phase`；信号处理；资源清理。
   - 方案：配置超时时间；实现 `@PreDestroy`；监听关闭事件。

---

## 实战配置题

1. **多数据源配置**
```java
@Configuration
public class MultiDataSourceConfig {
    
    @Primary
    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public PlatformTransactionManager transactionManager(
            @Qualifier("primaryDataSource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

2. **自定义 Starter 完整示例**
```java
// 1. 属性类
@ConfigurationProperties(prefix = "my.starter")
public class MyStarterProperties {
    private String name = "default";
    private int timeout = 1000;
    // getters and setters
}

// 2. 服务类
public class MyService {
    private final MyStarterProperties properties;
    
    public MyService(MyStarterProperties properties) {
        this.properties = properties;
    }
    
    public void doSomething() {
        // 业务逻辑
    }
}

// 3. 自动配置类
@Configuration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyStarterProperties.class)
public class MyAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyStarterProperties properties) {
        return new MyService(properties);
    }
}

// 4. spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration
```

3. **Actuator 安全配置**
```java
@Configuration
public class ActuatorSecurityConfig {
    
    @Bean
    public SecurityFilterChain actuatorSecurityFilterChain(HttpSecurity http) throws Exception {
        http.requestMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeRequests(requests -> requests
                .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
                .anyRequest().hasRole("ACTUATOR"))
            .httpBasic();
        return http.build();
    }
}
```

4. **异步配置**
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
    
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CustomAsyncExceptionHandler();
    }
}
```

5. **自定义 Banner**
```java
@SpringBootApplication
public class Application {
    
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        app.setBanner(new CustomBanner());
        app.run(args);
    }
}

class CustomBanner implements Banner {
    @Override
    public void printBanner(Environment environment, Class<?> sourceClass, PrintStream out) {
        out.println("=================================");
        out.println("    My Spring Boot Application  ");
        out.println("=================================");
    }
}
```

---

如果你需要我对内容进行精简版或加上更详细的示例答案，请告诉我你的偏好（例如"突出自动配置/启动流程""附带详细解答"）。


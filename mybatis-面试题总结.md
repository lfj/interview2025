# MyBatis 面试题总结

> 覆盖 MyBatis 核心原理、SQL 映射、动态 SQL、缓存机制、插件开发、Spring 集成、性能优化、常见问题与高频面试题。

## 目录
- [MyBatis 基础](#mybatis-基础)
- [核心组件与架构](#核心组件与架构)
- [SQL 映射](#sql-映射)
- [动态 SQL](#动态-sql)
- [缓存机制](#缓存机制)
- [插件机制](#插件机制)
- [与 Spring 集成](#与-spring-集成)
- [性能优化](#性能优化)
- [常见问题与解决方案](#常见问题与解决方案)
- [高频面试题](#高频面试题)

---

## MyBatis 基础

### 1. 什么是 MyBatis？

**MyBatis** 是一个优秀的**持久层框架**，它支持自定义 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。

**核心特点：**
- ✅ **SQL 可控**：可以编写原生 SQL，灵活度高
- ✅ **轻量级**：框架简单，学习成本低
- ✅ **半自动化**：需要手动编写 SQL，但结果映射自动化
- ✅ **ORM 映射**：支持对象关系映射
- ✅ **动态 SQL**：支持动态生成 SQL

**与其他框架对比：**

| 特性 | MyBatis | Hibernate | JPA |
|------|---------|-----------|-----|
| **SQL 控制** | 完全控制 | 自动生成 | 自动生成 |
| **学习曲线** | 平缓 | 陡峭 | 中等 |
| **性能** | 高（SQL 可控） | 中等 | 中等 |
| **灵活性** | 高 | 低 | 低 |
| **适用场景** | 复杂 SQL、性能要求高 | 简单 CRUD | 标准 JPA 规范 |

---

### 2. MyBatis vs Hibernate

**MyBatis 优势：**
- SQL 完全可控，可以优化复杂查询
- 学习成本低，上手快
- 性能好，适合高并发场景
- 适合复杂 SQL 和存储过程

**Hibernate 优势：**
- 完全自动化，无需编写 SQL
- 跨数据库支持好
- 对象关系映射强大
- 适合简单的 CRUD 操作

**选择建议：**
- **MyBatis**：复杂 SQL、性能要求高、需要 SQL 优化
- **Hibernate**：简单 CRUD、快速开发、跨数据库

---

### 3. MyBatis 核心概念

**1. SqlSession**
- MyBatis 的核心接口
- 代表与数据库的一次会话
- 线程不安全，不能共享

**2. Mapper 接口**
- 定义数据访问方法
- 通过动态代理实现
- 方法名对应 SQL 映射文件中的 id

**3. Mapper XML**
- 定义 SQL 语句
- 结果映射配置
- 参数映射配置

**4. Configuration**
- MyBatis 的配置信息
- 包含数据源、Mapper 注册等

---

## 核心组件与架构

### 1. MyBatis 架构图

```
[应用程序]
    ↓
[SqlSessionFactory]  ← 创建
    ↓
[SqlSession]  ← 获取
    ↓
[Executor]  ← 执行器
    ├─ SimpleExecutor（简单执行器）
    ├─ ReuseExecutor（重用执行器）
    └─ BatchExecutor（批量执行器）
    ↓
[StatementHandler]  ← 语句处理器
    ├─ PreparedStatementHandler
    ├─ CallableStatementHandler
    └─ SimpleStatementHandler
    ↓
[ParameterHandler]  ← 参数处理器
[ResultSetHandler]  ← 结果集处理器
    ↓
[TypeHandler]  ← 类型处理器
    ↓
[JDBC]
```

---

### 2. 核心组件详解

#### SqlSessionFactory

**作用：** 创建 SqlSession 的工厂类

**创建方式：**
```java
// 方式1：通过 XML 配置
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = 
    new SqlSessionFactoryBuilder().build(inputStream);

// 方式2：通过 Java 代码
DataSource dataSource = ...;
TransactionFactory transactionFactory = 
    new JdbcTransactionFactory();
Environment environment = 
    new Environment("development", transactionFactory, dataSource);
Configuration configuration = new Configuration(environment);
configuration.addMapper(UserMapper.class);
SqlSessionFactory sqlSessionFactory = 
    new SqlSessionFactoryBuilder().build(configuration);
```

**特点：**
- 线程安全，全局单例
- 创建成本高，应用启动时创建一次
- 包含所有配置信息

---

#### SqlSession

**作用：** 与数据库交互的会话对象

**主要方法：**
```java
// 查询
<T> T selectOne(String statement, Object parameter);
<E> List<E> selectList(String statement, Object parameter);

// 插入、更新、删除
int insert(String statement, Object parameter);
int update(String statement, Object parameter);
int delete(String statement, Object parameter);

// 获取 Mapper
<T> T getMapper(Class<T> type);

// 事务
void commit();
void rollback();
void close();
```

**使用方式：**
```java
// 方式1：直接使用 SqlSession
SqlSession sqlSession = sqlSessionFactory.openSession();
try {
    User user = sqlSession.selectOne(
        "com.example.mapper.UserMapper.selectById", 1L
    );
} finally {
    sqlSession.close();
}

// 方式2：使用 Mapper 接口（推荐）
SqlSession sqlSession = sqlSessionFactory.openSession();
try {
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    User user = mapper.selectById(1L);
} finally {
    sqlSession.close();
}
```

**线程安全性：**
- **SqlSession 不是线程安全的**
- 每个线程应该有自己的 SqlSession 实例
- 使用完及时关闭

---

#### Executor（执行器）

**作用：** 执行 SQL 语句的核心组件

**三种类型：**

**1. SimpleExecutor（简单执行器）**
```java
// 每次执行都创建新的 Statement
// 不重用 Statement
// 默认执行器
```

**2. ReuseExecutor（重用执行器）**
```java
// 重用 Statement
// 相同 SQL 只创建一次 Statement
// 适合频繁执行相同 SQL
```

**3. BatchExecutor（批量执行器）**
```java
// 批量执行 SQL
// 适合批量插入、更新、删除
// 需要手动调用 flushStatements()
```

**配置方式：**
```xml
<!-- mybatis-config.xml -->
<settings>
    <!-- 默认执行器类型 -->
    <setting name="defaultExecutorType" value="SIMPLE"/>
    <!-- 可选值：SIMPLE、REUSE、BATCH -->
</settings>
```

**使用示例：**
```java
// 批量执行
SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
try {
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    for (int i = 0; i < 1000; i++) {
        mapper.insert(new User("user" + i));
    }
    sqlSession.flushStatements();  // 批量提交
    sqlSession.commit();
} finally {
    sqlSession.close();
}
```

---

#### Mapper 接口与动态代理

**Mapper 接口定义：**
```java
public interface UserMapper {
    User selectById(Long id);
    List<User> selectAll();
    int insert(User user);
    int update(User user);
    int delete(Long id);
}
```

**动态代理实现原理：**
```java
// MyBatis 内部实现（简化版）
public class MapperProxy implements InvocationHandler {
    private SqlSession sqlSession;
    private Class<?> mapperInterface;
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) {
        // 1. 获取方法对应的 SQL 语句
        String statement = mapperInterface.getName() + "." + method.getName();
        
        // 2. 根据方法返回类型选择执行方法
        if (method.getReturnType() == List.class) {
            return sqlSession.selectList(statement, args[0]);
        } else {
            return sqlSession.selectOne(statement, args[0]);
        }
    }
}

// 创建代理对象
UserMapper mapper = (UserMapper) Proxy.newProxyInstance(
    UserMapper.class.getClassLoader(),
    new Class[]{UserMapper.class},
    new MapperProxy(sqlSession, UserMapper.class)
);
```

**Mapper 接口规则：**
- 方法名对应 Mapper XML 中的 `id`
- 方法参数对应 SQL 中的参数
- 方法返回类型对应结果映射

---

## SQL 映射

### 1. 参数映射

#### #{} vs ${}

**#{}（预编译参数）：**
```xml
<!-- 使用 #{} -->
<select id="selectById" resultType="User">
    SELECT * FROM users WHERE id = #{id}
</select>

<!-- 编译后的 SQL -->
SELECT * FROM users WHERE id = ?
<!-- 参数值：1 -->
```

**特点：**
- ✅ **防止 SQL 注入**：使用 PreparedStatement 预编译
- ✅ **自动类型转换**：MyBatis 自动处理类型
- ✅ **推荐使用**：安全、高效

**${}（字符串替换）：**
```xml
<!-- 使用 ${} -->
<select id="selectByTable" resultType="User">
    SELECT * FROM ${tableName}
</select>

<!-- 编译后的 SQL -->
SELECT * FROM users
<!-- 直接替换，不预编译 -->
```

**特点：**
- ⚠️ **SQL 注入风险**：直接字符串替换
- ⚠️ **需要手动处理**：需要自己处理特殊字符
- ✅ **动态表名/列名**：适合动态表名、列名场景

**使用场景：**
```xml
<!-- #{} 用于参数值 -->
<select id="selectById" resultType="User">
    SELECT * FROM users WHERE id = #{id}
</select>

<!-- ${} 用于表名、列名 -->
<select id="selectByTable" resultType="User">
    SELECT * FROM ${tableName} WHERE ${columnName} = #{value}
</select>
```

---

#### 参数传递方式

**1. 单个参数：**
```java
// Mapper 接口
User selectById(Long id);

// Mapper XML
<select id="selectById" resultType="User">
    SELECT * FROM users WHERE id = #{id}
</select>
```

**2. 多个参数（使用 @Param）：**
```java
// Mapper 接口
User selectByUsernameAndPassword(
    @Param("username") String username,
    @Param("password") String password
);

// Mapper XML
<select id="selectByUsernameAndPassword" resultType="User">
    SELECT * FROM users 
    WHERE username = #{username} AND password = #{password}
</select>
```

**3. 对象参数：**
```java
// Mapper 接口
int insert(User user);

// Mapper XML
<insert id="insert">
    INSERT INTO users (username, password, email)
    VALUES (#{username}, #{password}, #{email})
</insert>
```

**4. Map 参数：**
```java
// Mapper 接口
List<User> selectByMap(Map<String, Object> params);

// Mapper XML
<select id="selectByMap" resultType="User">
    SELECT * FROM users 
    WHERE username = #{username} AND age = #{age}
</select>
```

**5. 集合参数：**
```java
// Mapper 接口
List<User> selectByIds(@Param("ids") List<Long> ids);

// Mapper XML
<select id="selectByIds" resultType="User">
    SELECT * FROM users 
    WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```

---

### 2. 结果映射

#### resultType vs resultMap

**resultType（简单映射）：**
```xml
<!-- 自动映射：列名 → 属性名 -->
<select id="selectById" resultType="User">
    SELECT id, username, password, email FROM users WHERE id = #{id}
</select>

<!-- 要求：列名与属性名一致（或驼峰命名） -->
```

**resultMap（复杂映射）：**
```xml
<!-- 自定义映射规则 -->
<resultMap id="userResultMap" type="User">
    <id property="id" column="user_id"/>
    <result property="username" column="user_name"/>
    <result property="password" column="user_password"/>
    <result property="email" column="user_email"/>
</resultMap>

<select id="selectById" resultMap="userResultMap">
    SELECT user_id, user_name, user_password, user_email 
    FROM users 
    WHERE user_id = #{id}
</select>
```

---

#### 一对一映射（association）

```xml
<!-- User 包含 Order -->
<resultMap id="userWithOrderMap" type="User">
    <id property="id" column="user_id"/>
    <result property="username" column="username"/>
    <association property="order" javaType="Order">
        <id property="id" column="order_id"/>
        <result property="orderNo" column="order_no"/>
        <result property="amount" column="amount"/>
    </association>
</resultMap>

<select id="selectUserWithOrder" resultMap="userWithOrderMap">
    SELECT 
        u.id AS user_id,
        u.username,
        o.id AS order_id,
        o.order_no,
        o.amount
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    WHERE u.id = #{id}
</select>
```

**嵌套查询方式：**
```xml
<resultMap id="userWithOrderMap" type="User">
    <id property="id" column="id"/>
    <result property="username" column="username"/>
    <association 
        property="order" 
        column="id" 
        select="selectOrderByUserId"
        javaType="Order"/>
</resultMap>

<select id="selectUserWithOrder" resultMap="userWithOrderMap">
    SELECT * FROM users WHERE id = #{id}
</select>

<select id="selectOrderByUserId" resultType="Order">
    SELECT * FROM orders WHERE user_id = #{id}
</select>
```

---

#### 一对多映射（collection）

```xml
<!-- User 包含多个 Order -->
<resultMap id="userWithOrdersMap" type="User">
    <id property="id" column="user_id"/>
    <result property="username" column="username"/>
    <collection property="orders" ofType="Order">
        <id property="id" column="order_id"/>
        <result property="orderNo" column="order_no"/>
        <result property="amount" column="amount"/>
    </collection>
</resultMap>

<select id="selectUserWithOrders" resultMap="userWithOrdersMap">
    SELECT 
        u.id AS user_id,
        u.username,
        o.id AS order_id,
        o.order_no,
        o.amount
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    WHERE u.id = #{id}
</select>
```

**嵌套查询方式：**
```xml
<resultMap id="userWithOrdersMap" type="User">
    <id property="id" column="id"/>
    <result property="username" column="username"/>
    <collection 
        property="orders" 
        column="id" 
        select="selectOrdersByUserId"
        ofType="Order"/>
</resultMap>
```

---

#### 多对多映射

```xml
<!-- User 和 Role 多对多关系 -->
<resultMap id="userWithRolesMap" type="User">
    <id property="id" column="user_id"/>
    <result property="username" column="username"/>
    <collection property="roles" ofType="Role">
        <id property="id" column="role_id"/>
        <result property="roleName" column="role_name"/>
    </collection>
</resultMap>

<select id="selectUserWithRoles" resultMap="userWithRolesMap">
    SELECT 
        u.id AS user_id,
        u.username,
        r.id AS role_id,
        r.role_name
    FROM users u
    LEFT JOIN user_role ur ON u.id = ur.user_id
    LEFT JOIN roles r ON ur.role_id = r.id
    WHERE u.id = #{id}
</select>
```

---

### 3. 自动映射

**自动映射规则：**
- 列名 → 属性名（忽略大小写）
- 支持驼峰命名：`user_name` → `userName`

**配置：**
```xml
<settings>
    <!-- 自动映射级别 -->
    <!-- NONE：不自动映射 -->
    <!-- PARTIAL：自动映射（默认，不映射嵌套结果） -->
    <!-- FULL：完全自动映射（包括嵌套结果） -->
    <setting name="autoMappingBehavior" value="PARTIAL"/>
    
    <!-- 驼峰命名 -->
    <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
```

---

## 动态 SQL

### 1. if 标签

```xml
<select id="selectUsers" resultType="User">
    SELECT * FROM users
    <where>
        <if test="username != null and username != ''">
            AND username = #{username}
        </if>
        <if test="age != null">
            AND age = #{age}
        </if>
        <if test="email != null and email != ''">
            AND email = #{email}
        </if>
    </where>
</select>
```

---

### 2. choose、when、otherwise

```xml
<select id="selectUsers" resultType="User">
    SELECT * FROM users
    <where>
        <choose>
            <when test="username != null">
                AND username = #{username}
            </when>
            <when test="email != null">
                AND email = #{email}
            </when>
            <otherwise>
                AND status = 1
            </otherwise>
        </choose>
    </where>
</select>
```

---

### 3. where 标签

```xml
<!-- where 标签会自动处理 AND/OR，去除多余的 AND/OR -->
<select id="selectUsers" resultType="User">
    SELECT * FROM users
    <where>
        <if test="username != null">
            AND username = #{username}
        </if>
        <if test="age != null">
            AND age = #{age}
        </if>
    </where>
</select>

<!-- 等价于 -->
<select id="selectUsers" resultType="User">
    SELECT * FROM users
    WHERE 1=1
    <if test="username != null">
        AND username = #{username}
    </if>
    <if test="age != null">
        AND age = #{age}
    </if>
</select>
```

---

### 4. set 标签

```xml
<!-- set 标签会自动处理逗号，去除多余的逗号 -->
<update id="updateUser">
    UPDATE users
    <set>
        <if test="username != null">
            username = #{username},
        </if>
        <if test="password != null">
            password = #{password},
        </if>
        <if test="email != null">
            email = #{email},
        </if>
    </set>
    WHERE id = #{id}
</update>
```

---

### 5. foreach 标签

```xml
<!-- 批量查询 -->
<select id="selectByIds" resultType="User">
    SELECT * FROM users
    WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>

<!-- 批量插入 -->
<insert id="batchInsert">
    INSERT INTO users (username, password, email)
    VALUES
    <foreach collection="users" item="user" separator=",">
        (#{user.username}, #{user.password}, #{user.email})
    </foreach>
</insert>

<!-- 批量更新 -->
<update id="batchUpdate">
    <foreach collection="users" item="user" separator=";">
        UPDATE users
        SET username = #{user.username}, email = #{user.email}
        WHERE id = #{user.id}
    </foreach>
</update>
```

**foreach 属性：**
- `collection`：集合参数名
- `item`：集合中的元素变量名
- `index`：索引变量名
- `open`：开始符号
- `separator`：分隔符
- `close`：结束符号

---

### 6. trim 标签

```xml
<!-- trim 可以自定义前缀、后缀、去除的字符串 -->
<select id="selectUsers" resultType="User">
    SELECT * FROM users
    <trim prefix="WHERE" prefixOverrides="AND |OR ">
        <if test="username != null">
            AND username = #{username}
        </if>
        <if test="age != null">
            AND age = #{age}
        </if>
    </trim>
</select>
```

**trim 属性：**
- `prefix`：前缀
- `suffix`：后缀
- `prefixOverrides`：去除的前缀
- `suffixOverrides`：去除的后缀

---

### 7. bind 标签

```xml
<!-- bind 可以创建变量，用于模糊查询 -->
<select id="selectUsers" resultType="User">
    <bind name="pattern" value="'%' + username + '%'"/>
    SELECT * FROM users
    WHERE username LIKE #{pattern}
</select>
```

---

## 缓存机制

### 1. 一级缓存（SqlSession 级别）

**特点：**
- 作用域：SqlSession
- 默认开启，无法关闭
- 生命周期：SqlSession 关闭时清除

**工作原理：**
```java
SqlSession sqlSession = sqlSessionFactory.openSession();
try {
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    
    // 第一次查询：从数据库查询
    User user1 = mapper.selectById(1L);
    
    // 第二次查询：从一级缓存获取（不查询数据库）
    User user2 = mapper.selectById(1L);
    
    // user1 == user2（同一个对象）
} finally {
    sqlSession.close();
}
```

**缓存失效条件：**
1. SqlSession 关闭
2. 执行了 update、insert、delete 操作
3. 手动调用 `sqlSession.clearCache()`
4. 执行了 `sqlSession.commit()` 或 `sqlSession.rollback()`

---

### 2. 二级缓存（Mapper 级别）

**特点：**
- 作用域：Mapper 命名空间
- 默认关闭，需要手动开启
- 多个 SqlSession 共享
- 生命周期：应用关闭时清除

**配置方式：**

**1. 全局配置：**
```xml
<!-- mybatis-config.xml -->
<settings>
    <setting name="cacheEnabled" value="true"/>
</settings>
```

**2. Mapper XML 配置：**
```xml
<!-- UserMapper.xml -->
<mapper namespace="com.example.mapper.UserMapper">
    <!-- 开启二级缓存 -->
    <cache 
        eviction="LRU"
        flushInterval="60000"
        size="512"
        readOnly="true"/>
    
    <select id="selectById" resultType="User">
        SELECT * FROM users WHERE id = #{id}
    </select>
</mapper>
```

**cache 属性：**
- `eviction`：缓存回收策略（LRU、FIFO、SOFT、WEAK）
- `flushInterval`：刷新间隔（毫秒）
- `size`：缓存对象数量
- `readOnly`：是否只读

**使用示例：**
```java
// SqlSession 1
SqlSession sqlSession1 = sqlSessionFactory.openSession();
UserMapper mapper1 = sqlSession1.getMapper(UserMapper.class);
User user1 = mapper1.selectById(1L);  // 从数据库查询
sqlSession1.close();  // 提交到二级缓存

// SqlSession 2
SqlSession sqlSession2 = sqlSessionFactory.openSession();
UserMapper mapper2 = sqlSession2.getMapper(UserMapper.class);
User user2 = mapper2.selectById(1L);  // 从二级缓存获取
sqlSession2.close();
```

**缓存失效条件：**
1. 执行了 update、insert、delete 操作
2. 手动调用 `sqlSession.clearCache()`
3. 缓存达到最大容量（LRU 策略）

---

### 3. 缓存问题与解决方案

**问题 1：脏数据**
```java
// 场景：两个 SqlSession 同时操作
SqlSession sqlSession1 = sqlSessionFactory.openSession();
SqlSession sqlSession2 = sqlSessionFactory.openSession();

// SqlSession1 查询
UserMapper mapper1 = sqlSession1.getMapper(UserMapper.class);
User user1 = mapper1.selectById(1L);  // 缓存到二级缓存
sqlSession1.close();

// SqlSession2 更新
UserMapper mapper2 = sqlSession2.getMapper(UserMapper.class);
mapper2.updateUser(user1);  // 更新数据库
sqlSession2.commit();
sqlSession2.close();

// SqlSession3 查询（可能读到旧数据）
SqlSession sqlSession3 = sqlSessionFactory.openSession();
UserMapper mapper3 = sqlSession3.getMapper(UserMapper.class);
User user3 = mapper3.selectById(1L);  // 从二级缓存获取（可能是旧数据）
```

**解决方案：**
- 使用 Redis 等外部缓存
- 合理设置缓存过期时间
- 更新操作后清除缓存

**问题 2：序列化问题**
```xml
<!-- 如果 readOnly=false，对象需要序列化 -->
<cache readOnly="false"/>
```

```java
// User 类需要实现 Serializable
public class User implements Serializable {
    // ...
}
```

---

## 插件机制

### 1. 插件原理（拦截器）

**MyBatis 四大核心对象：**
- Executor（执行器）
- StatementHandler（语句处理器）
- ParameterHandler（参数处理器）
- ResultSetHandler（结果集处理器）

**插件可以拦截这些对象的方法：**

```java
@Intercepts({
    @Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
    )
})
public class MyPlugin implements Interceptor {
    
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 前置处理
        System.out.println("Before query");
        
        // 执行原方法
        Object result = invocation.proceed();
        
        // 后置处理
        System.out.println("After query");
        
        return result;
    }
    
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }
    
    @Override
    public void setProperties(Properties properties) {
        // 设置插件属性
    }
}
```

**注册插件：**
```xml
<!-- mybatis-config.xml -->
<plugins>
    <plugin interceptor="com.example.plugin.MyPlugin">
        <property name="property1" value="value1"/>
    </plugin>
</plugins>
```

---

### 2. 分页插件（PageHelper）

**使用方式：**
```java
// 1. 引入依赖
// 2. 配置插件
<plugins>
    <plugin interceptor="com.github.pagehelper.PageInterceptor">
        <property name="helperDialect" value="mysql"/>
    </plugin>
</plugins>

// 3. 使用
PageHelper.startPage(1, 10);  // 第1页，每页10条
List<User> users = userMapper.selectAll();
PageInfo<User> pageInfo = new PageInfo<>(users);
```

**原理：**
- 拦截 Executor 的 query 方法
- 在 SQL 前添加 `LIMIT` 子句
- 查询总数并封装到 PageInfo

---

### 3. 自定义插件示例

**SQL 执行时间统计插件：**
```java
@Intercepts({
    @Signature(
        type = Executor.class,
        method = "update",
        args = {MappedStatement.class, Object.class}
    ),
    @Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
    )
})
public class SqlExecuteTimePlugin implements Interceptor {
    
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = invocation.proceed();
            return result;
        } finally {
            long endTime = System.currentTimeMillis();
            long executeTime = endTime - startTime;
            
            MappedStatement mappedStatement = (MappedStatement) invocation.getArgs()[0];
            String sqlId = mappedStatement.getId();
            
            if (executeTime > 1000) {  // 超过1秒记录日志
                System.out.println("SQL执行时间过长: " + sqlId + ", 耗时: " + executeTime + "ms");
            }
        }
    }
    
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }
    
    @Override
    public void setProperties(Properties properties) {
        // 设置属性
    }
}
```

---

## 与 Spring 集成

### 1. Spring 集成配置

**方式 1：XML 配置**
```xml
<!-- applicationContext.xml -->
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="url" value="jdbc:mysql://localhost:3306/test"/>
    <property name="username" value="root"/>
    <property name="password" value="password"/>
</bean>

<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="mapperLocations" value="classpath:mapper/*.xml"/>
    <property name="configLocation" value="classpath:mybatis-config.xml"/>
</bean>

<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
    <property name="basePackage" value="com.example.mapper"/>
    <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
</bean>
```

**方式 2：Java 配置**
```java
@Configuration
public class MyBatisConfig {
    
    @Bean
    public DataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl("jdbc:mysql://localhost:3306/test");
        dataSource.setUsername("root");
        dataSource.setPassword("password");
        return dataSource;
    }
    
    @Bean
    public SqlSessionFactory sqlSessionFactory() throws Exception {
        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
        factoryBean.setDataSource(dataSource());
        factoryBean.setMapperLocations(
            new PathMatchingResourcePatternResolver()
                .getResources("classpath:mapper/*.xml")
        );
        return factoryBean.getObject();
    }
    
    @Bean
    public MapperScannerConfigurer mapperScannerConfigurer() {
        MapperScannerConfigurer configurer = new MapperScannerConfigurer();
        configurer.setBasePackage("com.example.mapper");
        configurer.setSqlSessionFactoryBeanName("sqlSessionFactory");
        return configurer;
    }
}
```

**方式 3：Spring Boot 配置**
```yaml
# application.yml
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.example.entity
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: true
```

```java
@SpringBootApplication
@MapperScan("com.example.mapper")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

### 2. 事务管理

**Spring 事务配置：**
```xml
<bean id="transactionManager" 
      class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>

<tx:annotation-driven transaction-manager="transactionManager"/>
```

```java
@Service
@Transactional
public class UserService {
    
    @Autowired
    private UserMapper userMapper;
    
    public void saveUser(User user) {
        userMapper.insert(user);
        // 如果抛出异常，事务会回滚
    }
}
```

---

## 性能优化

### 1. SQL 优化

**1. 避免 N+1 查询问题**

**问题示例：**
```xml
<!-- 会导致 N+1 查询 -->
<resultMap id="userWithOrdersMap" type="User">
    <id property="id" column="id"/>
    <collection property="orders" column="id" select="selectOrdersByUserId"/>
</resultMap>

<select id="selectUsers" resultMap="userWithOrdersMap">
    SELECT * FROM users
</select>
```

**解决方案：使用 JOIN 查询**
```xml
<resultMap id="userWithOrdersMap" type="User">
    <id property="id" column="user_id"/>
    <collection property="orders" ofType="Order">
        <id property="id" column="order_id"/>
    </collection>
</resultMap>

<select id="selectUsers" resultMap="userWithOrdersMap">
    SELECT 
        u.id AS user_id,
        o.id AS order_id
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
</select>
```

**2. 使用索引**
```sql
-- 确保 WHERE 条件字段有索引
SELECT * FROM users WHERE username = #{username}  -- username 需要索引
```

**3. 避免 SELECT ***
```xml
<!-- 不推荐 -->
<select id="selectUsers" resultType="User">
    SELECT * FROM users
</select>

<!-- 推荐 -->
<select id="selectUsers" resultType="User">
    SELECT id, username, email FROM users
</select>
```

---

### 2. 批量操作优化

**批量插入：**
```xml
<insert id="batchInsert">
    INSERT INTO users (username, password, email)
    VALUES
    <foreach collection="users" item="user" separator=",">
        (#{user.username}, #{user.password}, #{user.email})
    </foreach>
</insert>
```

**使用 BatchExecutor：**
```java
SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
try {
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    for (User user : users) {
        mapper.insert(user);
    }
    sqlSession.flushStatements();
    sqlSession.commit();
} finally {
    sqlSession.close();
}
```

---

### 3. 缓存优化

**1. 合理使用二级缓存**
- 读多写少的场景适合使用缓存
- 写多读少的场景不适合使用缓存

**2. 缓存失效策略**
- 更新操作后及时清除缓存
- 设置合理的缓存过期时间

---

### 4. 连接池优化

**Druid 连接池配置：**
```xml
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="initialSize" value="5"/>
    <property name="maxActive" value="20"/>
    <property name="minIdle" value="5"/>
    <property name="maxWait" value="60000"/>
    <property name="testOnBorrow" value="false"/>
    <property name="testWhileIdle" value="true"/>
    <property name="validationQuery" value="SELECT 1"/>
</bean>
```

---

## 常见问题与解决方案

### 1. 参数绑定问题

**问题：参数为 null**
```java
// Mapper 接口
User selectByUsername(String username);

// Mapper XML
<select id="selectByUsername" resultType="User">
    SELECT * FROM users WHERE username = #{username}
</select>

// 调用
userMapper.selectByUsername(null);  // 可能查询不到数据
```

**解决方案：**
```xml
<select id="selectByUsername" resultType="User">
    SELECT * FROM users
    <where>
        <if test="username != null">
            AND username = #{username}
        </if>
    </where>
</select>
```

---

### 2. 结果映射问题

**问题：列名与属性名不匹配**
```xml
<!-- 数据库：user_name，Java：username -->
<select id="selectUsers" resultType="User">
    SELECT user_name FROM users
</select>
```

**解决方案：**
```xml
<!-- 方案1：使用别名 -->
<select id="selectUsers" resultType="User">
    SELECT user_name AS username FROM users
</select>

<!-- 方案2：使用 resultMap -->
<resultMap id="userMap" type="User">
    <result property="username" column="user_name"/>
</resultMap>

<!-- 方案3：开启驼峰命名 -->
<setting name="mapUnderscoreToCamelCase" value="true"/>
```

---

### 3. 动态 SQL 问题

**问题：WHERE 条件拼接错误**
```xml
<!-- 错误示例 -->
<select id="selectUsers" resultType="User">
    SELECT * FROM users
    WHERE
    <if test="username != null">
        username = #{username}
    </if>
    <if test="age != null">
        AND age = #{age}  <!-- 如果 username 为 null，这里会多一个 AND -->
    </if>
</select>
```

**解决方案：使用 where 标签**
```xml
<select id="selectUsers" resultType="User">
    SELECT * FROM users
    <where>
        <if test="username != null">
            AND username = #{username}
        </if>
        <if test="age != null">
            AND age = #{age}
        </if>
    </where>
</select>
```

---

### 4. 事务问题

**问题：@Transactional 不生效**
```java
@Service
public class UserService {
    
    @Autowired
    private UserMapper userMapper;
    
    public void saveUser(User user) {
        // 同类方法调用，@Transactional 不生效
        insertUser(user);
    }
    
    @Transactional
    public void insertUser(User user) {
        userMapper.insert(user);
    }
}
```

**解决方案：**
```java
@Service
public class UserService {
    
    @Autowired
    private UserMapper userMapper;
    
    @Autowired
    private UserService userService;  // 注入自己
    
    public void saveUser(User user) {
        // 通过代理对象调用
        userService.insertUser(user);
    }
    
    @Transactional
    public void insertUser(User user) {
        userMapper.insert(user);
    }
}
```

---

### 5. 缓存问题

**问题：缓存数据不一致**
```java
// 更新数据后，缓存未更新
userMapper.updateUser(user);
// 下次查询可能还是旧数据
```

**解决方案：**
```java
// 更新后清除缓存
userMapper.updateUser(user);
sqlSession.clearCache();
```

---

## 高频面试题

### 1. MyBatis 的工作原理？

**答案：**
1. **加载配置**：读取 mybatis-config.xml 和 Mapper XML
2. **创建 SqlSessionFactory**：通过配置创建工厂
3. **创建 SqlSession**：通过工厂创建会话
4. **获取 Mapper**：通过动态代理获取 Mapper 接口实现
5. **执行 SQL**：Executor 执行 SQL，StatementHandler 处理语句
6. **结果映射**：ResultSetHandler 处理结果集，映射到 Java 对象
7. **返回结果**：返回查询结果

---

### 2. #{} 和 ${} 的区别？

**答案：**
- **#{}**：预编译参数，使用 PreparedStatement，防止 SQL 注入，自动类型转换
- **${}**：字符串替换，直接拼接 SQL，有 SQL 注入风险，需要手动处理

**使用场景：**
- **#{}**：用于参数值（推荐）
- **${}**：用于表名、列名等动态部分

---

### 3. MyBatis 的一级缓存和二级缓存？

**答案：**

**一级缓存：**
- 作用域：SqlSession
- 默认开启，无法关闭
- 生命周期：SqlSession 关闭时清除

**二级缓存：**
- 作用域：Mapper 命名空间
- 默认关闭，需要手动开启
- 多个 SqlSession 共享
- 需要序列化支持

---

### 4. MyBatis 如何防止 SQL 注入？

**答案：**
- 使用 **#{}** 预编译参数，而不是 **${}** 字符串替换
- MyBatis 使用 PreparedStatement，参数会被转义
- 避免直接拼接 SQL 语句

---

### 5. MyBatis 的 Mapper 接口如何工作？

**答案：**
- MyBatis 使用**动态代理**创建 Mapper 接口实现
- 代理对象拦截方法调用
- 根据方法名和参数找到对应的 SQL 语句
- 执行 SQL 并返回结果

---

### 6. MyBatis 如何处理结果映射？

**答案：**
- **resultType**：简单映射，列名与属性名一致
- **resultMap**：复杂映射，自定义映射规则
- 支持一对一（association）、一对多（collection）映射
- 支持自动映射和手动映射

---

### 7. MyBatis 的动态 SQL 有哪些标签？

**答案：**
- **if**：条件判断
- **choose、when、otherwise**：多条件选择
- **where**：WHERE 子句处理
- **set**：SET 子句处理
- **foreach**：循环处理
- **trim**：自定义前缀后缀
- **bind**：创建变量

---

### 8. MyBatis 的插件机制？

**答案：**
- 基于**拦截器（Interceptor）**实现
- 可以拦截四大核心对象：Executor、StatementHandler、ParameterHandler、ResultSetHandler
- 使用 JDK 动态代理实现
- 常见应用：分页插件、SQL 执行时间统计

---

### 9. MyBatis 如何与 Spring 集成？

**答案：**
- 使用 `SqlSessionFactoryBean` 创建 SqlSessionFactory
- 使用 `MapperScannerConfigurer` 扫描 Mapper 接口
- 使用 `@MapperScan` 注解扫描 Mapper
- Spring Boot 自动配置 MyBatis

---

### 10. MyBatis 的性能优化？

**答案：**
- **SQL 优化**：避免 N+1 查询、使用索引、避免 SELECT *
- **批量操作**：使用 BatchExecutor、批量插入
- **缓存优化**：合理使用二级缓存
- **连接池优化**：配置合适的连接池参数
- **结果映射优化**：避免复杂的嵌套查询

---

### 11. MyBatis 如何处理分页？

**答案：**
- **物理分页**：使用 LIMIT（推荐）
- **逻辑分页**：使用 RowBounds（不推荐，内存占用大）
- **分页插件**：PageHelper（推荐）

---

### 12. MyBatis 的事务管理？

**答案：**
- MyBatis 本身不管理事务
- 需要配合 Spring 事务管理
- 使用 `@Transactional` 注解
- 支持声明式事务和编程式事务

---

### 13. MyBatis 如何处理延迟加载？

**答案：**
- 使用 `lazyLoadingEnabled=true` 开启延迟加载
- 使用 `fetchType="lazy"` 指定延迟加载
- 延迟加载使用 CGLIB 动态代理实现

---

### 14. MyBatis 的 Executor 类型？

**答案：**
- **SimpleExecutor**：简单执行器，每次创建新 Statement
- **ReuseExecutor**：重用执行器，重用 Statement
- **BatchExecutor**：批量执行器，批量执行 SQL

---

### 15. MyBatis 如何处理多数据源？

**答案：**
- 配置多个 SqlSessionFactory
- 使用 `@DS` 注解（MyBatis-Plus）
- 使用 AOP 动态切换数据源
- 使用 AbstractRoutingDataSource

---

## 总结

MyBatis 是一个优秀的持久层框架，核心特点：
- SQL 可控，灵活度高
- 学习成本低，上手快
- 性能好，适合高并发
- 支持动态 SQL、缓存、插件等高级特性

**适用场景：**
- 复杂 SQL 查询
- 性能要求高的场景
- 需要 SQL 优化的项目
- 团队熟悉 SQL 的项目

**不适用场景：**
- 简单的 CRUD（可以考虑 JPA）
- 需要跨数据库的项目（可以考虑 Hibernate）
- 完全不想写 SQL 的项目

---

如需我补充更多内容或示例，请告知你的需求。



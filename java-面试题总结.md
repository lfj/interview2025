# Java 面试题总结（Markdown 版）

> 覆盖基础语法、面向对象、集合框架、多线程并发、JVM、设计模式、Spring生态、数据库、分布式、微服务、性能优化、故障排查。高频考点与追问路径，含示例代码与最佳实践。

## 目录
- [基础语法与特性](#基础语法与特性)
- [面向对象编程](#面向对象编程)
- [集合框架](#集合框架)
- [多线程与并发](#多线程与并发)
- [I/O 模型](#io-模型)
- [JVM 原理与调优](#jvm-原理与调优)
- [设计模式](#设计模式)
- [Spring 框架](#spring-框架)
- [数据库与持久化](#数据库与持久化)
- [分布式与微服务](#分布式与微服务)
- [消息队列](#消息队列)
- [缓存](#缓存)
- [性能优化](#性能优化)
- [故障排查](#故障排查)
- [常见场景题](#常见场景题)
- [实战编程题](#实战编程题)

---

## 基础语法与特性

1. **Java 8 新特性（Lambda、Stream、Optional）**
   - 要点：函数式编程、方法引用、Stream 操作（map/filter/reduce/collect）、Optional 避免 NPE。
   - 追问：Stream 并行流与线程安全；Optional 滥用场景。

2. **String、StringBuilder、StringBuffer 区别**
   - 要点：String 不可变；StringBuilder 非线程安全但性能好；StringBuffer 线程安全。
   - 追问：字符串拼接性能；`intern()` 方法的作用域。

3. **== 与 equals() 的区别**
   - 要点：`==` 比较引用；`equals()` 比较内容（需重写 `hashCode()`）。
   - 追问：`hashCode()` 与 `equals()` 的契约；为什么重写 `equals()` 必须重写 `hashCode()`。

4. **基本类型与包装类型**
   - 要点：自动装箱/拆箱；缓存机制（-128 到 127）；`Integer.valueOf()` vs `new Integer()`。
   - 追问：性能影响；NPE 风险。

5. **final、finally、finalize**
   - 要点：`final` 修饰类/方法/变量；`finally` 异常处理；`finalize()` 已废弃。
   - 追问：`final` 对性能的影响；`try-with-resources` 替代 `finally`。

6. **异常体系**
   - 要点：`Throwable` → `Error`/`Exception`；`RuntimeException` vs 检查异常；异常处理最佳实践。
   - 追问：异常吞没与日志；自定义异常设计。

### 示例：Stream API 使用
```java
// 分组统计
Map<String, Long> countByDept = employees.stream()
    .filter(e -> e.getAge() > 25)
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));

// 并行流注意线程安全
List<Integer> result = list.parallelStream()
    .map(x -> x * 2)
    .collect(Collectors.toList());
```

---

## 面向对象编程

1. **封装、继承、多态**
   - 要点：访问修饰符；方法重写与重载；运行时多态（虚方法表）。
   - 追问：多态的实现原理；方法分派（静态/动态）。

2. **抽象类 vs 接口**
   - 要点：Java 8 接口支持默认方法；抽象类可以有构造器与字段；接口多继承。
   - 追问：设计选择；`default` 方法冲突解决。

3. **内部类**
   - 要点：成员内部类、局部内部类、匿名内部类、静态内部类；访问外部变量需 `final`。
   - 追问：为什么局部变量需 `final`；内存泄漏风险。

4. **反射机制**
   - 要点：`Class`、`Method`、`Field`；动态代理（JDK Proxy、CGLIB）。
   - 追问：性能影响；Spring AOP 实现原理。

5. **泛型**
   - 要点：类型擦除；通配符（`? extends T`、`? super T`）；PECS 原则。
   - 追问：为什么不能 `new T[]`；类型擦除带来的限制。

### 示例：动态代理
```java
// JDK 动态代理
public class ProxyExample {
    public static void main(String[] args) {
        InvocationHandler handler = (proxy, method, args) -> {
            System.out.println("Before method: " + method.getName());
            Object result = method.invoke(target, args);
            System.out.println("After method: " + method.getName());
            return result;
        };
        MyInterface proxy = (MyInterface) Proxy.newProxyInstance(
            MyInterface.class.getClassLoader(),
            new Class[]{MyInterface.class},
            handler
        );
    }
}
```

---

## 集合框架

1. **ArrayList vs LinkedList**
   - 要点：ArrayList 基于数组，随机访问 O(1)，插入删除 O(n)；LinkedList 基于双向链表，插入删除 O(1)。
   - 追问：扩容机制；`modCount` 与 `ConcurrentModificationException`。

2. **HashMap 原理**
   - 要点：数组 + 链表/红黑树；哈希冲突处理；扩容机制（2 倍，rehash）；JDK 1.8 优化。
   - 追问：为什么容量是 2 的幂；`hash()` 方法的作用；线程安全问题。

3. **ConcurrentHashMap**
   - 要点：JDK 1.7 分段锁；JDK 1.8 CAS + synchronized；`sizeCtl` 控制扩容。
   - 追问：为什么读不加锁；`transfer()` 扩容过程。

4. **HashSet vs TreeSet**
   - 要点：HashSet 基于 HashMap；TreeSet 基于 TreeMap（红黑树），有序。
   - 追问：`compareTo()` 与 `equals()` 一致性。

5. **Fail-Fast vs Fail-Safe**
   - 要点：ArrayList/HashMap 迭代器 `modCount` 检查；CopyOnWriteArrayList 快照迭代。
   - 追问：并发修改异常场景；`ConcurrentModificationException` 触发条件。

### 示例：HashMap 扩容
```java
// JDK 1.8 HashMap 扩容关键代码逻辑
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // 阈值翻倍
    }
    // ... 创建新数组并 rehash
}
```

---

## 多线程与并发

1. **线程创建方式**
   - 要点：继承 `Thread`、实现 `Runnable`、实现 `Callable`、线程池。
   - 追问：为什么推荐线程池；`Future` vs `CompletableFuture`。

2. **synchronized 原理**
   - 要点：对象头 Mark Word；偏向锁 → 轻量级锁 → 重量级锁；锁升级过程。
   - 追问：`monitorenter`/`monitorexit`；锁消除与锁粗化。

3. **volatile 关键字**
   - 要点：可见性（MESI 协议）；禁止指令重排序（内存屏障）；不保证原子性。
   - 追问：`happens-before` 规则；单例模式双重检查锁定。

4. **CAS 与 ABA 问题**
   - 要点：`Unsafe` 类；`AtomicInteger` 实现；ABA 问题与 `AtomicStampedReference`。
   - 追问：CAS 自旋开销；`LongAdder` vs `AtomicLong`。

5. **AQS（AbstractQueuedSynchronizer）**
   - 要点：CLH 队列；`state` 状态；`ReentrantLock`、`CountDownLatch`、`Semaphore` 基于 AQS。
   - 追问：公平锁 vs 非公平锁；`tryAcquire()` 实现。

6. **线程池（ThreadPoolExecutor）**
   - 要点：核心参数（corePoolSize、maximumPoolSize、workQueue、rejectedExecutionHandler）；执行流程。
   - 追问：`ThreadPoolExecutor` vs `Executors`；拒绝策略选择；线程池监控。

7. **并发工具类**
   - 要点：`CountDownLatch`（一次性）、`CyclicBarrier`（可重用）、`Semaphore`（信号量）、`Exchanger`。
   - 追问：适用场景；与 `CompletableFuture` 对比。

### 示例：线程池使用
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5,                              // corePoolSize
    10,                             // maximumPoolSize
    60L,                            // keepAliveTime
    TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100), // workQueue
    new ThreadFactoryBuilder()
        .setNameFormat("worker-%d")
        .build(),
    new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略
);

// 提交任务
Future<String> future = executor.submit(() -> {
    // 业务逻辑
    return "result";
});
```

### 示例：生产者消费者
```java
// 使用 BlockingQueue
BlockingQueue<String> queue = new LinkedBlockingQueue<>(10);

// 生产者
producer = () -> {
    while (true) {
        queue.put("item");
    }
};

// 消费者
consumer = () -> {
    while (true) {
        String item = queue.take();
        // 处理
    }
};
```

---

## I/O 模型

1. **BIO、NIO、AIO 的区别**
   - 要点：BIO（阻塞 I/O）、NIO（非阻塞 I/O）、AIO（异步 I/O）；适用场景；性能对比。
   - 追问：为什么需要 NIO；Selector 的作用；零拷贝原理。

### I/O 模型详解

#### 什么是 I/O？

I/O（Input/Output）是程序与外部世界（文件、网络、设备等）进行数据交换的过程。

**I/O 操作的两个阶段：**
1. **等待数据就绪**：等待数据从网络/磁盘到达内核缓冲区
2. **数据拷贝**：将数据从内核缓冲区拷贝到用户空间

#### Java 的三种 I/O 模型

| I/O 模型 | 全称 | 阻塞情况 | 线程模型 | 适用场景 |
|---------|------|---------|---------|---------|
| **BIO** | Blocking I/O（阻塞 I/O） | 完全阻塞 | 一连接一线程 | 连接数少、长连接 |
| **NIO** | Non-blocking I/O（非阻塞 I/O） | 非阻塞 | 多路复用 | 高并发、短连接 |
| **AIO** | Asynchronous I/O（异步 I/O） | 异步 | 回调机制 | 高并发、大文件 |

---

### 1. BIO（Blocking I/O）阻塞 I/O

#### 什么是 BIO？

BIO 是**同步阻塞 I/O**，当程序执行 I/O 操作时，线程会被阻塞，直到数据准备好或操作完成。

**特点：**
- ✅ 简单易用，编程模型直观
- ✅ 适合连接数少的场景
- ❌ 一个连接需要一个线程
- ❌ 线程资源消耗大
- ❌ 不适合高并发

#### BIO 工作流程

```
客户端请求
    ↓
服务器创建线程处理
    ↓
线程阻塞等待数据（read/write）
    ↓
数据到达，处理请求
    ↓
返回响应，线程继续等待下一个请求
```

#### BIO 代码示例

**服务器端：**

```java
// BIO 服务器
public class BIOServer {
    
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("BIO 服务器启动，端口：8080");
        
        while (true) {
            // 阻塞：等待客户端连接
            Socket socket = serverSocket.accept();
            System.out.println("新客户端连接：" + socket.getRemoteSocketAddress());
            
            // 为每个连接创建新线程
            new Thread(() -> {
                try {
                    handleClient(socket);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
    
    private static void handleClient(Socket socket) throws IOException {
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(socket.getInputStream())
        );
        PrintWriter writer = new PrintWriter(
            socket.getOutputStream(), true
        );
        
        String line;
        // 阻塞：等待客户端发送数据
        while ((line = reader.readLine()) != null) {
            System.out.println("收到消息：" + line);
            writer.println("服务器回复：" + line);
        }
        
        socket.close();
    }
}
```

**客户端：**

```java
// BIO 客户端
public class BIOClient {
    
    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("localhost", 8080);
        
        PrintWriter writer = new PrintWriter(socket.getOutputStream(), true);
        BufferedReader reader = new BufferedReader(
            new InputStreamReader(socket.getInputStream())
        );
        
        writer.println("Hello Server");
        
        // 阻塞：等待服务器响应
        String response = reader.readLine();
        System.out.println("服务器响应：" + response);
        
        socket.close();
    }
}
```

#### BIO 的问题

**问题 1：线程资源消耗**

```
1000 个并发连接 = 1000 个线程
每个线程占用：
- 栈内存：1MB
- 总内存：1000MB = 1GB
- CPU 上下文切换开销巨大
```

**问题 2：阻塞等待**

```java
// 线程在这里阻塞，什么都不能做
socket.read();  // 等待数据，线程被阻塞
// 即使没有数据，线程也不能处理其他任务
```

**问题 3：扩展性差**

- 线程数受限于操作系统
- 无法支持大量并发连接
- 资源浪费严重

---

### 2. NIO（Non-blocking I/O）非阻塞 I/O

#### 什么是 NIO？

NIO 是**同步非阻塞 I/O**，程序执行 I/O 操作时不会阻塞，可以立即返回，通过轮询或事件通知机制处理数据。

**核心组件：**
- **Channel**：通道，类似流但可以双向传输
- **Buffer**：缓冲区，数据容器
- **Selector**：选择器，多路复用器

**特点：**
- ✅ 非阻塞，线程不会等待
- ✅ 一个线程可以处理多个连接
- ✅ 适合高并发场景
- ❌ 编程复杂度较高
- ❌ 需要理解 Channel、Buffer、Selector

#### NIO 工作流程

```
客户端请求
    ↓
注册到 Selector
    ↓
Selector 轮询检查就绪的 Channel
    ↓
有数据就绪 → 处理
无数据就绪 → 继续轮询（不阻塞）
    ↓
处理完成后继续监听
```

#### NIO 核心概念

**1. Channel（通道）**

Channel 是双向的，可以读也可以写。

```java
// 主要 Channel 类型
ServerSocketChannel  // 服务器端通道
SocketChannel       // 客户端通道
FileChannel         // 文件通道
DatagramChannel     // UDP 通道
```

**2. Buffer（缓冲区）**

Buffer 是数据容器，用于在 Channel 和程序之间传输数据。

```java
// Buffer 类型
ByteBuffer
CharBuffer
IntBuffer
LongBuffer
// ...

// Buffer 的三个重要属性
capacity  // 容量
position  // 当前位置
limit     // 限制位置
```

**3. Selector（选择器）**

Selector 是 NIO 的核心，用于监听多个 Channel 的事件。

```java
// Selector 监听的事件
SelectionKey.OP_ACCEPT   // 接受连接
SelectionKey.OP_CONNECT   // 连接就绪
SelectionKey.OP_READ      // 读就绪
SelectionKey.OP_WRITE     // 写就绪
```

#### NIO 代码示例

**服务器端：**

```java
// NIO 服务器
public class NIOServer {
    
    public static void main(String[] args) throws IOException {
        // 1. 创建 Selector
        Selector selector = Selector.open();
        
        // 2. 创建 ServerSocketChannel
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.socket().bind(new InetSocketAddress(8080));
        serverChannel.configureBlocking(false);  // 设置为非阻塞
        
        // 3. 注册到 Selector，监听 ACCEPT 事件
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        
        System.out.println("NIO 服务器启动，端口：8080");
        
        while (true) {
            // 4. 阻塞等待事件（可以设置超时）
            int readyChannels = selector.select(1000);  // 1秒超时
            
            if (readyChannels == 0) {
                continue;  // 没有事件，继续轮询
            }
            
            // 5. 获取就绪的 Channel
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = selectedKeys.iterator();
            
            while (keyIterator.hasNext()) {
                SelectionKey key = keyIterator.next();
                
                if (key.isAcceptable()) {
                    // 接受新连接
                    handleAccept(selector, key);
                } else if (key.isReadable()) {
                    // 读取数据
                    handleRead(key);
                }
                
                keyIterator.remove();  // 处理完后移除
            }
        }
    }
    
    private static void handleAccept(Selector selector, SelectionKey key) 
            throws IOException {
        ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
        
        // 接受连接
        SocketChannel clientChannel = serverChannel.accept();
        clientChannel.configureBlocking(false);
        
        // 注册读事件
        clientChannel.register(selector, SelectionKey.OP_READ);
        
        System.out.println("新客户端连接：" + clientChannel.getRemoteAddress());
    }
    
    private static void handleRead(SelectionKey key) throws IOException {
        SocketChannel channel = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        
        int bytesRead = channel.read(buffer);
        
        if (bytesRead > 0) {
            buffer.flip();  // 切换为读模式
            byte[] bytes = new byte[buffer.remaining()];
            buffer.get(bytes);
            
            String message = new String(bytes);
            System.out.println("收到消息：" + message);
            
            // 回复客户端
            ByteBuffer response = ByteBuffer.wrap(
                ("服务器回复：" + message).getBytes()
            );
            channel.write(response);
        } else if (bytesRead < 0) {
            // 连接关闭
            channel.close();
        }
    }
}
```

**客户端：**

```java
// NIO 客户端
public class NIOClient {
    
    public static void main(String[] args) throws IOException {
        SocketChannel channel = SocketChannel.open();
        channel.configureBlocking(false);
        channel.connect(new InetSocketAddress("localhost", 8080));
        
        // 等待连接完成
        while (!channel.finishConnect()) {
            // 可以做其他事情
        }
        
        // 发送数据
        ByteBuffer buffer = ByteBuffer.wrap("Hello Server".getBytes());
        channel.write(buffer);
        
        // 读取响应
        buffer = ByteBuffer.allocate(1024);
        while (channel.read(buffer) == 0) {
            // 没有数据，继续等待
        }
        
        buffer.flip();
        byte[] bytes = new byte[buffer.remaining()];
        buffer.get(bytes);
        System.out.println("服务器响应：" + new String(bytes));
        
        channel.close();
    }
}
```

#### NIO 的优势

**1. 非阻塞**

```java
// BIO：阻塞等待
socket.read();  // 线程被阻塞

// NIO：非阻塞
int bytes = channel.read(buffer);
if (bytes == 0) {
    // 没有数据，可以做其他事情
    // 线程不会被阻塞
}
```

**2. 多路复用**

```java
// 一个线程可以处理多个连接
Selector selector = Selector.open();
// 注册多个 Channel
channel1.register(selector, SelectionKey.OP_READ);
channel2.register(selector, SelectionKey.OP_READ);
channel3.register(selector, SelectionKey.OP_READ);

// 一个线程监听所有 Channel
selector.select();  // 返回就绪的 Channel
```

**3. 零拷贝（部分支持）**

```java
// 使用 FileChannel 的 transferTo 实现零拷贝
FileChannel sourceChannel = new FileInputStream("source.txt").getChannel();
FileChannel destChannel = new FileOutputStream("dest.txt").getChannel();

// 零拷贝：直接从内核缓冲区传输，不经过用户空间
sourceChannel.transferTo(0, sourceChannel.size(), destChannel);
```

#### NIO 的问题

**1. 编程复杂**

- 需要理解 Channel、Buffer、Selector
- 需要处理各种边界情况
- 代码量多

**2. 空轮询 Bug**

```java
// Selector.select() 可能立即返回，即使没有事件
// 导致 CPU 100% 占用
while (true) {
    selector.select();  // 可能立即返回 0
    // CPU 空转
}
```

**3. 仍然需要轮询**

- 虽然非阻塞，但仍需要轮询检查
- 不是真正的异步

---

### 3. AIO（Asynchronous I/O）异步 I/O

#### 什么是 AIO？

AIO 是**异步非阻塞 I/O**，程序发起 I/O 操作后立即返回，操作系统完成 I/O 操作后通过回调通知程序。

**特点：**
- ✅ 真正的异步，无需轮询
- ✅ 回调机制，编程相对简单
- ✅ 适合高并发、大文件操作
- ❌ Windows 支持好，Linux 支持有限
- ❌ 在 Linux 上性能可能不如 NIO

#### AIO 工作流程

```
程序发起 I/O 操作
    ↓
立即返回（不阻塞）
    ↓
操作系统处理 I/O
    ↓
I/O 完成 → 回调通知
    ↓
程序处理结果
```

#### AIO 核心组件

**1. AsynchronousServerSocketChannel**

```java
// 异步服务器通道
AsynchronousServerSocketChannel serverChannel = 
    AsynchronousServerSocketChannel.open();
```

**2. AsynchronousSocketChannel**

```java
// 异步客户端通道
AsynchronousSocketChannel clientChannel = 
    AsynchronousSocketChannel.open();
```

**3. CompletionHandler**

```java
// 完成处理器（回调接口）
CompletionHandler<V, A> {
    void completed(V result, A attachment);  // 成功回调
    void failed(Throwable exc, A attachment); // 失败回调
}
```

#### AIO 代码示例

**服务器端：**

```java
// AIO 服务器
public class AIOServer {
    
    public static void main(String[] args) throws IOException, InterruptedException {
        // 1. 创建异步服务器通道
        AsynchronousServerSocketChannel serverChannel = 
            AsynchronousServerSocketChannel.open();
        
        // 2. 绑定端口
        serverChannel.bind(new InetSocketAddress(8080));
        
        System.out.println("AIO 服务器启动，端口：8080");
        
        // 3. 异步接受连接
        serverChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
            
            @Override
            public void completed(AsynchronousSocketChannel clientChannel, Void attachment) {
                // 接受连接成功，继续接受下一个连接
                serverChannel.accept(null, this);
                
                // 处理客户端连接
                handleClient(clientChannel);
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                System.err.println("接受连接失败：" + exc);
            }
        });
        
        // 主线程等待
        Thread.sleep(Long.MAX_VALUE);
    }
    
    private static void handleClient(AsynchronousSocketChannel clientChannel) {
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        
        // 异步读取数据
        clientChannel.read(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>() {
            
            @Override
            public void completed(Integer result, ByteBuffer buffer) {
                if (result > 0) {
                    buffer.flip();
                    byte[] bytes = new byte[buffer.remaining()];
                    buffer.get(bytes);
                    
                    String message = new String(bytes);
                    System.out.println("收到消息：" + message);
                    
                    // 异步写入响应
                    ByteBuffer response = ByteBuffer.wrap(
                        ("服务器回复：" + message).getBytes()
                    );
                    clientChannel.write(response, response, new CompletionHandler<Integer, ByteBuffer>() {
                        @Override
                        public void completed(Integer result, ByteBuffer buffer) {
                            // 写入成功，继续读取
                            ByteBuffer newBuffer = ByteBuffer.allocate(1024);
                            clientChannel.read(newBuffer, newBuffer, this);
                        }
                        
                        @Override
                        public void failed(Throwable exc, ByteBuffer buffer) {
                            try {
                                clientChannel.close();
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        }
                    });
                } else if (result < 0) {
                    // 连接关闭
                    try {
                        clientChannel.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
            
            @Override
            public void failed(Throwable exc, ByteBuffer buffer) {
                try {
                    clientChannel.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

**客户端：**

```java
// AIO 客户端
public class AIOClient {
    
    public static void main(String[] args) throws IOException, InterruptedException {
        AsynchronousSocketChannel channel = AsynchronousSocketChannel.open();
        
        // 异步连接
        channel.connect(new InetSocketAddress("localhost", 8080), null, 
            new CompletionHandler<Void, Void>() {
                
                @Override
                public void completed(Void result, Void attachment) {
                    // 连接成功，发送数据
                    ByteBuffer buffer = ByteBuffer.wrap("Hello Server".getBytes());
                    channel.write(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>() {
                        
                        @Override
                        public void completed(Integer result, ByteBuffer buffer) {
                            // 发送成功，读取响应
                            ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                            channel.read(readBuffer, readBuffer, 
                                new CompletionHandler<Integer, ByteBuffer>() {
                                    
                                    @Override
                                    public void completed(Integer result, ByteBuffer buffer) {
                                        buffer.flip();
                                        byte[] bytes = new byte[buffer.remaining()];
                                        buffer.get(bytes);
                                        System.out.println("服务器响应：" + new String(bytes));
                                        
                                        try {
                                            channel.close();
                                        } catch (IOException e) {
                                            e.printStackTrace();
                                        }
                                    }
                                    
                                    @Override
                                    public void failed(Throwable exc, ByteBuffer buffer) {
                                        exc.printStackTrace();
                                    }
                                });
                        }
                        
                        @Override
                        public void failed(Throwable exc, ByteBuffer buffer) {
                            exc.printStackTrace();
                        }
                    });
                }
                
                @Override
                public void failed(Throwable exc, Void attachment) {
                    exc.printStackTrace();
                }
            });
        
        // 等待异步操作完成
        Thread.sleep(2000);
    }
}
```

#### AIO 的优势

**1. 真正的异步**

```java
// 发起 I/O 操作后立即返回
channel.read(buffer, buffer, handler);
// 线程不会被阻塞，可以做其他事情

// I/O 完成后，操作系统通过回调通知
handler.completed(result, buffer);
```

**2. 回调机制**

- 不需要轮询
- 操作系统通知程序
- 编程模型清晰

**3. 适合大文件操作**

```java
// 异步文件 I/O
AsynchronousFileChannel fileChannel = 
    AsynchronousFileChannel.open(path, StandardOpenOption.READ);

ByteBuffer buffer = ByteBuffer.allocate(1024);
fileChannel.read(buffer, 0, buffer, new CompletionHandler<Integer, ByteBuffer>() {
    @Override
    public void completed(Integer result, ByteBuffer buffer) {
        // 读取完成
    }
    
    @Override
    public void failed(Throwable exc, ByteBuffer buffer) {
        // 读取失败
    }
});
```

#### AIO 的问题

**1. Linux 支持有限**

- Linux 的 AIO 实现不完善
- 底层仍使用 epoll（类似 NIO）
- 性能优势不明显

**2. 编程复杂**

- 回调嵌套深
- 错误处理复杂
- 代码可读性差

**3. 适用场景有限**

- 文件 I/O 效果好
- 网络 I/O 在 Linux 上不如 NIO

---

### 三种 I/O 模型对比

| 特性 | BIO | NIO | AIO |
|------|-----|-----|-----|
| **阻塞情况** | 完全阻塞 | 非阻塞 | 异步非阻塞 |
| **线程模型** | 一连接一线程 | 多路复用 | 回调机制 |
| **编程复杂度** | 简单 | 中等 | 复杂 |
| **适用场景** | 连接数少 | 高并发 | 大文件、高并发 |
| **性能** | 低 | 高 | 高（文件 I/O） |
| **操作系统支持** | 所有 | 所有 | Windows 好，Linux 有限 |
| **典型应用** | 传统服务器 | Netty、Tomcat NIO | 文件服务器 |

### 实际应用场景

**1. BIO 适用场景**
- 连接数少（< 100）
- 长连接
- 简单应用

**2. NIO 适用场景**
- 高并发（> 1000 连接）
- 短连接
- 网络服务器（Netty、Tomcat NIO）

**3. AIO 适用场景**
- 大文件操作
- 高并发文件服务器
- Windows 平台

### 总结

**关键理解：**

1. **BIO**：同步阻塞，简单但性能差
2. **NIO**：同步非阻塞，适合高并发，需要理解 Selector
3. **AIO**：异步非阻塞，真正的异步，但 Linux 支持有限

**选择建议：**
- **大多数场景**：使用 NIO（Netty）
- **简单应用**：使用 BIO
- **大文件操作**：考虑 AIO
- **高并发网络**：NIO + Reactor 模式（如 Netty）

---

## JVM 原理与调优

1. **内存模型（堆、栈、方法区）**
   - 要点：堆（新生代/老年代）、栈（局部变量表/操作数栈）、方法区（元空间）、程序计数器、本地方法栈。
   - 追问：为什么方法区移到元空间；栈溢出与堆溢出。

2. **垃圾回收算法**
   - 要点：标记-清除、标记-复制、标记-整理；分代收集理论。
   - 追问：各算法优缺点；为什么新生代用复制算法。

3. **垃圾回收器**
   - 要点：Serial、Parallel、CMS、G1、ZGC、Shenandoah；适用场景。
   - 追问：CMS 三色标记与并发失败；G1 的 Mixed GC；ZGC 低延迟原理。

4. **GC 调优**
   - 要点：`-Xms/-Xmx`、`-XX:NewRatio`、`-XX:SurvivorRatio`、`-XX:MaxGCPauseMillis`。
   - 追问：如何定位 Full GC；GC 日志分析；内存泄漏排查。

5. **类加载机制**
   - 要点：加载 → 验证 → 准备 → 解析 → 初始化；双亲委派模型；打破双亲委派（SPI、OSGi）。
   - 追问：为什么需要双亲委派；`ClassNotFoundException` vs `NoClassDefFoundError`。

### 类加载流程详解

#### 一、类加载的五个阶段

Java 类加载过程分为五个阶段：**加载（Loading）→ 验证（Verification）→ 准备（Preparation）→ 解析（Resolution）→ 初始化（Initialization）**。

```
[.class 文件]
    ↓
[加载 Loading]      → 将 .class 文件加载到内存
    ↓
[验证 Verification]  → 验证字节码的正确性
    ↓
[准备 Preparation]  → 为类变量分配内存并设置默认值
    ↓
[解析 Resolution]   → 将符号引用转换为直接引用
    ↓
[初始化 Initialization] → 执行类构造器 <clinit>() 方法
    ↓
[类可以使用]
```

---

#### 1. 加载（Loading）

**定义：** 将类的字节码文件（.class）从磁盘加载到内存中，并创建一个 `Class` 对象。

**具体操作：**
- 通过类的全限定名获取字节流（从文件系统、网络、ZIP 包、JAR 包等）
- 将字节流转换为方法区的运行时数据结构
- 在内存中生成一个代表该类的 `java.lang.Class` 对象，作为方法区中该类各种数据的访问入口

**示例：**
```java
// 加载 User 类
Class<?> clazz = Class.forName("com.example.User");
// 此时 User.class 文件被加载到内存，创建 Class 对象
```

**加载来源：**
- 本地文件系统
- ZIP/JAR 包
- 网络（Applet）
- 运行时计算生成（动态代理）
- 其他文件（JSP、数据库等）

---

#### 2. 验证（Verification）

**定义：** 确保被加载的类的字节码符合 JVM 规范，不会危害虚拟机安全。

**验证内容：**

**① 文件格式验证**
- 是否以魔数 `0xCAFEBABE` 开头
- 主次版本号是否在 JVM 处理范围内
- 常量池中的常量类型是否正确
- 各种索引值是否指向正确的常量

**② 元数据验证**
- 类是否有父类（除了 Object）
- 是否继承了不允许继承的类（final 类）
- 是否实现了接口中的所有方法
- 字段、方法是否与父类冲突

**③ 字节码验证**
- 数据流和控制流分析
- 确保方法体中的类型转换有效
- 确保跳转指令不会跳转到方法体外的字节码

**④ 符号引用验证**
- 符号引用中的类、字段、方法是否存在
- 访问权限是否合法（private、protected 等）

**示例：**
```java
// 如果字节码被篡改，验证阶段会抛出 VerifyError
// 例如：方法返回类型不匹配、访问不存在的字段等
```

---

#### 3. 准备（Preparation）

**定义：** 为类的静态变量分配内存并设置默认初始值（零值）。

**重要说明：**
- **只分配内存，不执行赋值**
- **只处理类变量（static），不处理实例变量**
- **设置的是默认值，不是代码中的初始值**

**默认值规则：**
```java
// 准备阶段的值
static int a;           // 0（不是代码中的初始值）
static long b;          // 0L
static boolean c;       // false
static Object d;        // null
static String e;         // null

// 注意：如果类变量有 final 修饰，且是编译期常量
static final int f = 123;  // 准备阶段直接赋值为 123（编译期常量）
static final String g = "hello";  // 准备阶段直接赋值为 "hello"
```

**示例：**
```java
public class Test {
    // 准备阶段：为 value 分配内存，设置为 0（不是 10）
    private static int value = 10;
    
    // 准备阶段：为 name 分配内存，设置为 null（不是 "test"）
    private static String name = "test";
    
    // 准备阶段：直接赋值为 100（编译期常量）
    private static final int CONSTANT = 100;
}
```

---

#### 4. 解析（Resolution）

**定义：** 将常量池中的符号引用替换为直接引用。

**符号引用 vs 直接引用：**

**符号引用（Symbolic Reference）：**
- 以一组符号描述所引用的目标
- 可以是任何形式的字面量
- 与虚拟机内存布局无关
- 例如：`com/example/User`、`java/lang/Object`

**直接引用（Direct Reference）：**
- 直接指向目标的指针、相对偏移量或能间接定位到目标的句柄
- 与虚拟机内存布局相关
- 例如：内存地址、方法表的索引

**解析内容：**
- **类或接口的解析**：将类名解析为直接引用
- **字段解析**：解析字段的符号引用
- **方法解析**：解析方法的符号引用
- **接口方法解析**：解析接口方法的符号引用

**示例：**
```java
// 符号引用：com.example.User
// 解析后：直接引用（内存地址）

public class Test {
    // 解析 User 类的符号引用
    private User user;
    
    // 解析 User.getName() 方法的符号引用
    public void test() {
        user.getName();  // 解析方法引用
    }
}
```

**解析时机：**
- 可以在加载阶段之后立即解析（早期解析）
- 也可以在符号引用被使用前才解析（延迟解析）

---

#### 5. 初始化（Initialization）

**定义：** 执行类的构造器 `<clinit>()` 方法，初始化类变量和静态代码块。

**重要说明：**
- **执行 `<clinit>()` 方法**（类构造器，不是实例构造器）
- **初始化类变量和静态代码块**
- **父类的 `<clinit>()` 先于子类执行**
- **`<clinit>()` 方法是线程安全的**（JVM 保证只有一个线程执行）

**`<clinit>()` 方法特点：**
```java
public class Test {
    // 这些代码会被编译到 <clinit>() 方法中
    private static int a = 10;        // ①
    private static int b;
    
    static {                          // ②
        b = 20;
        System.out.println("静态代码块执行");
    }
    
    private static int c = 30;        // ③
    
    // <clinit>() 方法执行顺序：① → ② → ③
    // 执行结果：a=10, b=20, c=30
}
```

**初始化时机（触发条件）：**
1. 创建类的实例（`new`）
2. 访问类的静态变量（非 final）
3. 调用类的静态方法
4. 使用反射（`Class.forName()`）
5. 初始化子类时，父类会先初始化
6. 虚拟机启动时，主类（包含 `main` 方法的类）会被初始化

**示例：**
```java
public class Parent {
    static {
        System.out.println("Parent 初始化");
    }
    public static int value = 100;
}

public class Child extends Parent {
    static {
        System.out.println("Child 初始化");
    }
}

public class Test {
    public static void main(String[] args) {
        // 触发 Child 初始化
        // 输出：
        // Parent 初始化
        // Child 初始化
        System.out.println(Child.value);
    }
}
```

**不会触发初始化的场景：**
```java
// 1. 通过子类引用父类的静态字段，不会触发子类初始化
System.out.println(Child.value);  // 只初始化 Parent

// 2. 通过数组定义引用类，不会触发初始化
Child[] array = new Child[10];  // 不会初始化 Child

// 3. 引用编译期常量，不会触发初始化
public class Test {
    public static final String CONSTANT = "hello";  // 编译期常量
}
System.out.println(Test.CONSTANT);  // 不会初始化 Test
```

---

#### 二、类加载器（ClassLoader）

**类加载器的层次结构：**

```
Bootstrap ClassLoader (启动类加载器)
    ↓
Extension ClassLoader (扩展类加载器)
    ↓
Application ClassLoader (应用程序类加载器)
    ↓
Custom ClassLoader (自定义类加载器)
```

**1. Bootstrap ClassLoader（启动类加载器）**
- 由 C++ 实现，是 JVM 的一部分
- 负责加载 `JAVA_HOME/lib` 目录下的核心类库
- 例如：`java.lang.*`、`java.util.*` 等
- 无法被 Java 程序直接引用

**2. Extension ClassLoader（扩展类加载器）**
- 由 `sun.misc.Launcher$ExtClassLoader` 实现
- 负责加载 `JAVA_HOME/lib/ext` 目录下的扩展类库
- 例如：`javax.*` 等

**3. Application ClassLoader（应用程序类加载器）**
- 由 `sun.misc.Launcher$AppClassLoader` 实现
- 负责加载用户类路径（ClassPath）下的类
- 是程序中默认的类加载器

**4. Custom ClassLoader（自定义类加载器）**
- 用户自定义的类加载器
- 继承 `ClassLoader` 类
- 可以指定加载路径

**示例：获取类加载器**
```java
public class ClassLoaderTest {
    public static void main(String[] args) {
        // 获取当前类的类加载器
        ClassLoader loader = ClassLoaderTest.class.getClassLoader();
        System.out.println(loader);  // sun.misc.Launcher$AppClassLoader
        
        // 获取父类加载器
        System.out.println(loader.getParent());  // sun.misc.Launcher$ExtClassLoader
        
        // 获取父类的父类（Bootstrap ClassLoader）
        System.out.println(loader.getParent().getParent());  // null（C++ 实现）
        
        // String 类由 Bootstrap ClassLoader 加载
        System.out.println(String.class.getClassLoader());  // null
    }
}
```

---

#### 三、双亲委派模型（Parent Delegation Model）

**定义：** 当一个类加载器收到类加载请求时，它不会自己先加载，而是委托给父类加载器去加载，只有当父类加载器无法加载时，子加载器才会尝试加载。

**工作流程：**
```
1. 收到类加载请求
    ↓
2. 委托给父类加载器
    ↓
3. 父类加载器再委托给它的父类
    ↓
4. 一直向上委托到 Bootstrap ClassLoader
    ↓
5. Bootstrap 尝试加载
    ↓
6. 如果加载失败，向下返回，让子加载器尝试
    ↓
7. 如果所有父加载器都失败，当前加载器才尝试加载
```

**代码实现（简化版）：**
```java
protected Class<?> loadClass(String name, boolean resolve) {
    // 1. 先检查类是否已经被加载
    Class<?> c = findLoadedClass(name);
    if (c == null) {
        try {
            if (parent != null) {
                // 2. 委托给父类加载器
                c = parent.loadClass(name, false);
            } else {
                // 3. 父类为 null，委托给 Bootstrap ClassLoader
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {
            // 父类加载器加载失败
        }
        
        if (c == null) {
            // 4. 父类加载器无法加载，自己尝试加载
            c = findClass(name);
        }
    }
    
    if (resolve) {
        resolveClass(c);
    }
    return c;
}
```

**为什么需要双亲委派？**

**1. 安全性**
```java
// 防止用户自定义的 java.lang.String 类替换核心类
// 因为 Bootstrap ClassLoader 会先加载核心类
// 用户自定义的类永远不会被加载
```

**2. 避免重复加载**
```java
// 同一个类只会被加载一次
// 由最顶层的类加载器加载，保证类的唯一性
```

**3. 保证核心类库的优先级**
```java
// 核心类库由 Bootstrap ClassLoader 加载
// 不会被用户自定义的类替换
```

**示例：双亲委派的作用**
```java
// 用户尝试定义 java.lang.String（会被阻止）
package java.lang;

public class String {  // 编译可以通过，但运行时会报错
    // SecurityException: Prohibited package name: java.lang
}
```

---

#### 四、打破双亲委派模型

**为什么需要打破？**

在某些场景下，需要打破双亲委派模型：
- **SPI（Service Provider Interface）**：JDBC、JNDI 等
- **OSGi**：模块化框架
- **热部署**：需要重新加载类

**如何打破？**

**方法 1：重写 `loadClass()` 方法**
```java
// 不委托给父类，直接自己加载
@Override
protected Class<?> loadClass(String name, boolean resolve) {
    // 不调用 super.loadClass()，直接自己加载
    return findClass(name);
}
```

**方法 2：使用线程上下文类加载器（Thread Context ClassLoader）**
```java
// SPI 机制使用线程上下文类加载器
// 例如：JDBC 驱动加载
Class.forName("com.mysql.jdbc.Driver", true, 
    Thread.currentThread().getContextClassLoader());
```

**SPI 示例：JDBC 驱动加载**
```java
// JDBC 使用 SPI 机制打破双亲委派
// java.sql.Driver 由 Bootstrap ClassLoader 加载
// 但 MySQL 驱动由 Application ClassLoader 加载
// 通过线程上下文类加载器解决

// 1. 设置线程上下文类加载器
Thread.currentThread().setContextClassLoader(
    ClassLoader.getSystemClassLoader()
);

// 2. ServiceLoader 使用线程上下文类加载器加载驱动
ServiceLoader<Driver> drivers = ServiceLoader.load(Driver.class);
```

---

#### 五、常见问题

**1. ClassNotFoundException vs NoClassDefFoundError**

**ClassNotFoundException：**
- 发生在**类加载阶段**
- 找不到类的定义文件（.class）
- 是**检查异常**（Checked Exception）
- 常见原因：类路径错误、类文件缺失

```java
// 示例
Class.forName("com.example.NonExistentClass");
// 抛出：ClassNotFoundException
```

**NoClassDefFoundError：**
- 发生在**类链接阶段**（验证、准备、解析）
- 类加载时找到了，但链接时找不到
- 是**错误**（Error），不是异常
- 常见原因：类加载成功，但依赖的类找不到

```java
// 示例
public class A {
    private B b = new B();  // B 类在编译时存在，但运行时找不到
}
// 抛出：NoClassDefFoundError
```

**2. 类加载的线程安全性**

- **加载阶段**：可以并发加载不同的类
- **初始化阶段**：`<clinit>()` 方法是线程安全的，JVM 保证只有一个线程执行
- **同一个类**：只会被加载一次

**3. 类的卸载**

- 类可以被卸载，但条件苛刻：
  - 该类的所有实例都被回收
  - 加载该类的 ClassLoader 被回收
  - 该类对应的 Class 对象没有被引用

**4. 自定义类加载器示例**

```java
public class CustomClassLoader extends ClassLoader {
    private String classPath;
    
    public CustomClassLoader(String classPath) {
        this.classPath = classPath;
    }
    
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            // 读取 .class 文件
            byte[] data = loadClassData(name);
            // 定义类
            return defineClass(name, data, 0, data.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
    
    private byte[] loadClassData(String name) throws IOException {
        String path = classPath + "/" + name.replace(".", "/") + ".class";
        FileInputStream fis = new FileInputStream(path);
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        int len;
        while ((len = fis.read()) != -1) {
            baos.write(len);
        }
        fis.close();
        return baos.toByteArray();
    }
}
```

---

#### 六、类加载流程图总结

```
[触发类加载]
    ↓
[双亲委派模型]
    ↓
[加载 Loading]
    ├─ 获取字节流
    ├─ 转换为方法区数据结构
    └─ 创建 Class 对象
    ↓
[验证 Verification]
    ├─ 文件格式验证
    ├─ 元数据验证
    ├─ 字节码验证
    └─ 符号引用验证
    ↓
[准备 Preparation]
    └─ 为静态变量分配内存并设置默认值
    ↓
[解析 Resolution]
    └─ 符号引用 → 直接引用
    ↓
[初始化 Initialization]
    └─ 执行 <clinit>() 方法
    ↓
[类可以使用]
```

---

#### 七、面试要点总结

1. **五个阶段**：加载 → 验证 → 准备 → 解析 → 初始化
2. **准备阶段**：只分配内存，设置默认值，不执行赋值
3. **初始化时机**：new、访问静态变量、调用静态方法、反射、子类初始化、主类
4. **双亲委派**：安全性、避免重复加载、保证核心类优先级
5. **打破双亲委派**：SPI、OSGi、热部署
6. **异常区别**：`ClassNotFoundException`（加载时）vs `NoClassDefFoundError`（链接时）

6. **JIT 编译**
   - 要点：解释执行 → C1 编译 → C2 编译；热点代码检测；`-XX:CompileThreshold`。
   - 追问：为什么需要解释器；分层编译策略。

### 示例：JVM 参数
```bash
# 堆内存设置
-Xms2g -Xmx4g

# 新生代比例
-XX:NewRatio=2

# Survivor 区比例
-XX:SurvivorRatio=8

# GC 日志
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/path/to/gc.log

# G1 参数
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m
```

---

## 设计模式

1. **单例模式**
   - 要点：饿汉式、懒汉式（双重检查锁定）、静态内部类、枚举；线程安全。
   - 追问：为什么枚举单例最安全；序列化问题。

2. **工厂模式**
   - 要点：简单工厂、工厂方法、抽象工厂；解耦创建与使用。
   - 追问：Spring BeanFactory vs ApplicationContext。

3. **代理模式**
   - 要点：静态代理、动态代理（JDK、CGLIB）；AOP 实现基础。
   - 追问：JDK Proxy vs CGLIB；性能差异。

4. **观察者模式**
   - 要点：`Observer`/`Observable`（已废弃）；事件驱动；Spring 事件机制。
   - 追问：与发布-订阅模式区别。

5. **策略模式**
   - 要点：算法族封装；`Comparator` 是典型应用。
   - 追问：与工厂模式结合；消除 if-else。

6. **模板方法模式**
   - 要点：定义算法骨架；`AbstractList`、`HttpServlet`。
   - 追问：钩子方法；与策略模式区别。

### 示例：单例模式（枚举）
```java
public enum Singleton {
    INSTANCE;
    
    public void doSomething() {
        // 业务逻辑
    }
}
```

### 示例：策略模式
```java
// 策略接口
interface PaymentStrategy {
    void pay(int amount);
}

// 具体策略
class AlipayStrategy implements PaymentStrategy {
    @Override
    public void pay(int amount) {
        System.out.println("支付宝支付: " + amount);
    }
}

// 上下文
class PaymentContext {
    private PaymentStrategy strategy;
    
    public PaymentContext(PaymentStrategy strategy) {
        this.strategy = strategy;
    }
    
    public void executePayment(int amount) {
        strategy.pay(amount);
    }
}
```

---

## Spring 框架

1. **IoC 容器**
   - 要点：控制反转；依赖注入（构造器/Setter/字段注入）；Bean 生命周期。
   - 追问：循环依赖解决（三级缓存）；`@Autowired` vs `@Resource`。

2. **AOP 原理**
   - 要点：代理模式；JDK Proxy vs CGLIB；切点表达式；通知类型（Before/After/Around）。
   - 追问：`@Transactional` 失效场景；AOP 执行顺序。

3. **Spring Bean 作用域**
   - 要点：singleton、prototype、request、session、application；线程安全问题。
   - 追问：为什么 Controller 建议用 singleton；prototype 使用场景。

4. **Spring 事务管理**
   - 要点：声明式事务；传播行为（REQUIRED、REQUIRES_NEW、NESTED）；隔离级别。
   - 追问：事务失效原因；`@Transactional` 在同类方法调用不生效。

5. **Spring MVC 流程**
   - 要点：DispatcherServlet → HandlerMapping → HandlerAdapter → ViewResolver。
   - 追问：`@Controller` vs `@RestController`；参数绑定原理。

6. **Spring Boot 自动配置**
   - 要点：`@EnableAutoConfiguration`；`spring.factories`；条件注解（`@ConditionalOnClass`）。
   - 追问：如何排除自动配置；自定义 Starter。

### 示例：循环依赖解决
```java
// Spring 三级缓存解决循环依赖
// 一级缓存：singletonObjects（完整 Bean）
// 二级缓存：earlySingletonObjects（早期引用）
// 三级缓存：singletonFactories（ObjectFactory）

// 流程：A 依赖 B，B 依赖 A
// 1. 创建 A，放入三级缓存
// 2. 发现依赖 B，创建 B
// 3. B 发现依赖 A，从三级缓存获取 A 的 ObjectFactory，得到早期引用
// 4. B 创建完成，A 注入 B
// 5. A 创建完成，移除三级缓存，放入一级缓存
```

### 示例：事务传播行为
```java
@Service
public class UserService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void methodA() {
        // 如果 methodB 抛出异常，methodA 也会回滚
        methodB();
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() {
        // 新事务，独立提交/回滚
    }
}
```

---

## 数据库与持久化

1. **MyBatis vs Hibernate**
   - 要点：MyBatis SQL 可控；Hibernate ORM 自动化；性能与灵活性权衡。
   - 追问：N+1 查询问题；一级/二级缓存。

2. **数据库连接池**
   - 要点：HikariCP、Druid、C3P0；连接泄漏检测；参数调优。
   - 追问：为什么需要连接池；`maxLifetime` vs `idleTimeout`。

3. **SQL 优化**
   - 要点：索引（B+树、最左前缀、覆盖索引）；执行计划；慢查询分析。
   - 追问：索引失效场景；分页优化（深分页问题）。

4. **事务隔离级别**
   - 要点：READ UNCOMMITTED、READ COMMITTED、REPEATABLE READ、SERIALIZABLE；MVCC。
   - 追问：脏读/幻读/不可重复读；MySQL InnoDB 默认级别。

5. **分布式事务**
   - 要点：2PC、3PC、TCC、Saga、Seata；CAP 理论。
   - 追问：最终一致性；补偿机制。

### 示例：MyBatis 动态 SQL
```xml
<select id="findUsers" resultType="User">
    SELECT * FROM users
    <where>
        <if test="name != null">
            AND name = #{name}
        </if>
        <if test="age != null">
            AND age = #{age}
        </if>
    </where>
    <choose>
        <when test="orderBy == 'name'">
            ORDER BY name
        </when>
        <otherwise>
            ORDER BY id
        </otherwise>
    </choose>
</select>
```

---

## 分布式与微服务

1. **CAP 理论**
   - 要点：一致性、可用性、分区容错性；只能满足两个；BASE 理论。
   - 追问：不同场景的选择；最终一致性实现。

2. **服务注册与发现**
   - 要点：Eureka、Consul、Nacos；AP vs CP；健康检查。
   - 追问：服务下线延迟；多级缓存。

3. **负载均衡**
   - 要点：轮询、随机、加权、一致性哈希；Ribbon、Spring Cloud LoadBalancer。
   - 追问：雪崩效应；熔断降级。

4. **分布式锁**
   - 要点：Redis（SETNX + EXPIRE）、ZooKeeper、数据库；Redisson 实现。
   - 追问：锁续期；可重入锁；死锁避免。

5. **分布式 ID**
   - 要点：UUID、雪花算法、数据库自增、Redis 自增、Leaf。
   - 追问：时钟回拨问题；全局唯一性保证。

6. **API 网关**
   - 要点：路由、限流、鉴权、熔断；Spring Cloud Gateway、Zuul。
   - 追问：网关性能；统一异常处理。

### 示例：Redis 分布式锁
```java
// Redisson 实现
RLock lock = redisson.getLock("myLock");
try {
    // 尝试加锁，最多等待 100 秒，锁定后 10 秒自动解锁
    boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
    if (res) {
        // 业务逻辑
    }
} catch (InterruptedException e) {
    e.printStackTrace();
} finally {
    lock.unlock();
}
```

---

## 消息队列

1. **消息队列作用**
   - 要点：解耦、异步、削峰；可靠性保证。
   - 追问：消息丢失场景；重复消费处理。

2. **RabbitMQ**
   - 要点：Exchange、Queue、Binding；工作模式（简单/工作队列/发布订阅/路由/主题）。
   - 追问：消息持久化；死信队列；集群模式。

3. **Kafka**
   - 要点：Topic、Partition、Consumer Group；高吞吐原理（顺序写、零拷贝、批量发送）。
   - 追问：消息顺序性；分区策略；`acks` 参数。

4. **RocketMQ**
   - 要点：NameServer、Broker、Producer、Consumer；事务消息；顺序消息。
   - 追问：消息过滤；延迟消息。

### 示例：Kafka 生产者配置
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("acks", "all"); // 等待所有副本确认
props.put("retries", 3);
props.put("batch.size", 16384);
props.put("linger.ms", 1);
props.put("buffer.memory", 33554432);
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
```

---

## 缓存

1. **缓存策略**
   - 要点：Cache Aside、Read/Write Through、Write Back；缓存穿透/击穿/雪崩。
   - 追问：解决方案（布隆过滤器、互斥锁、多级缓存）。

2. **Redis 数据结构**
   - 要点：String、Hash、List、Set、Sorted Set；底层实现（SDS、跳跃表、压缩列表）。
   - 追问：为什么 Redis 快；持久化（RDB、AOF）。

3. **Redis 集群**
   - 要点：主从复制、哨兵、Cluster；数据分片（哈希槽）。
   - 追问：脑裂问题；一致性哈希 vs 哈希槽。

4. **缓存更新策略**
   - 要点：先更新数据库再删除缓存；延迟双删；最终一致性。
   - 追问：缓存与数据库一致性；缓存预热。

### 示例：缓存穿透解决方案
```java
// 布隆过滤器 + 缓存空值
public User getUser(Long id) {
    // 1. 布隆过滤器判断
    if (!bloomFilter.mightContain(id)) {
        return null;
    }
    
    // 2. 查询缓存
    String key = "user:" + id;
    User user = redisTemplate.opsForValue().get(key);
    if (user != null) {
        return user == NULL_OBJECT ? null : user;
    }
    
    // 3. 查询数据库
    user = userMapper.selectById(id);
    if (user == null) {
        // 缓存空值，防止穿透
        redisTemplate.opsForValue().set(key, NULL_OBJECT, 5, TimeUnit.MINUTES);
        return null;
    }
    
    // 4. 写入缓存
    redisTemplate.opsForValue().set(key, user, 30, TimeUnit.MINUTES);
    return user;
}
```

---

## 性能优化

1. **代码层面优化**
   - 要点：减少对象创建；使用 StringBuilder；合理使用集合；避免反射。
   - 追问：字符串拼接性能；`ArrayList` vs `LinkedList` 选择。

2. **JVM 调优**
   - 要点：堆内存设置；GC 选择；GC 参数调优；JIT 编译优化。
   - 追问：如何定位内存泄漏；Full GC 频繁原因。

3. **数据库优化**
   - 要点：索引优化；SQL 优化；分库分表；读写分离。
   - 追问：慢查询分析；深分页优化。

4. **缓存优化**
   - 要点：缓存命中率；缓存预热；多级缓存；缓存更新策略。
   - 追问：缓存穿透/击穿/雪崩；热点数据识别。

### 示例：深分页优化
```sql
-- 低效：OFFSET 越大越慢
SELECT * FROM users ORDER BY id LIMIT 10000, 20;

-- 优化1：使用子查询
SELECT * FROM users 
WHERE id > (SELECT id FROM users ORDER BY id LIMIT 10000, 1)
ORDER BY id LIMIT 20;

-- 优化2：延迟关联
SELECT u.* FROM users u
INNER JOIN (SELECT id FROM users ORDER BY id LIMIT 10000, 20) t
ON u.id = t.id;
```

---

## 故障排查

1. **CPU 飙升**
   - 要点：`top`/`jstack` 定位线程；死循环/频繁 GC；火焰图分析。
   - 方案：定位热点方法；优化算法；减少 GC。

2. **内存泄漏**
   - 要点：`jmap` 导出堆；`jhat`/MAT 分析；常见原因（集合未清理、监听器未移除）。
   - 方案：修复代码；增加监控；定期 Full GC。

3. **线程死锁**
   - 要点：`jstack` 查看线程状态；`ThreadMXBean` 检测死锁。
   - 方案：避免嵌套锁；使用超时锁；统一锁顺序。

4. **OOM 排查**
   - 要点：堆溢出/栈溢出/方法区溢出；`-XX:+HeapDumpOnOutOfMemoryError`。
   - 方案：分析堆转储；调整 JVM 参数；修复代码。

### 示例：死锁检测
```java
ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();
if (deadlockedThreads != null) {
    ThreadInfo[] threadInfos = threadMXBean.getThreadInfo(deadlockedThreads);
    for (ThreadInfo threadInfo : threadInfos) {
        System.out.println(threadInfo);
    }
}
```

---

## 常见场景题

1. **接口响应慢**
   - 要点：数据库慢查询、缓存未命中、外部服务超时、GC 频繁。
   - 方案：SQL 优化、增加缓存、超时设置、JVM 调优。

2. **系统突然卡顿**
   - 要点：Full GC、死锁、数据库连接池耗尽、线程池满。
   - 方案：查看 GC 日志、线程 dump、连接池监控、扩容。

3. **数据不一致**
   - 要点：缓存与数据库不一致、分布式事务问题、并发更新。
   - 方案：延迟双删、最终一致性、乐观锁/悲观锁。

4. **消息重复消费**
   - 要点：网络重试、Consumer 重启、分区重平衡。
   - 方案：幂等性设计、消息去重、事务消息。

5. **缓存穿透**
   - 要点：查询不存在的数据、缓存未命中、大量请求打数据库。
   - 方案：布隆过滤器、缓存空值、参数校验。

---

## 实战编程题

1. **手写单例模式（线程安全）**
```java
public class Singleton {
    private volatile static Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

2. **手写 LRU 缓存**
```java
class LRUCache {
    class Node {
        int key, value;
        Node prev, next;
        Node(int k, int v) { key = k; value = v; }
    }
    
    private Map<Integer, Node> map = new HashMap<>();
    private Node head, tail;
    private int capacity;
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
    }
    
    public int get(int key) {
        Node node = map.get(key);
        if (node == null) return -1;
        moveToHead(node);
        return node.value;
    }
    
    public void put(int key, int value) {
        Node node = map.get(key);
        if (node != null) {
            node.value = value;
            moveToHead(node);
        } else {
            if (map.size() >= capacity) {
                removeTail();
            }
            node = new Node(key, value);
            map.put(key, node);
            addToHead(node);
        }
    }
    
    private void moveToHead(Node node) {
        removeNode(node);
        addToHead(node);
    }
    
    private void addToHead(Node node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }
    
    private void removeNode(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
    
    private void removeTail() {
        Node last = tail.prev;
        removeNode(last);
        map.remove(last.key);
    }
}
```

3. **手写生产者消费者**
```java
class ProducerConsumer {
    private final BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(10);
    
    class Producer implements Runnable {
        @Override
        public void run() {
            try {
                for (int i = 0; i < 100; i++) {
                    queue.put(i);
                    System.out.println("Produced: " + i);
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
    
    class Consumer implements Runnable {
        @Override
        public void run() {
            try {
                while (true) {
                    Integer item = queue.take();
                    System.out.println("Consumed: " + item);
                    Thread.sleep(200);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

4. **手写线程池**
```java
class SimpleThreadPool {
    private final BlockingQueue<Runnable> workQueue;
    private final List<WorkerThread> threads = new ArrayList<>();
    private volatile boolean shutdown = false;
    
    public SimpleThreadPool(int poolSize, int queueSize) {
        workQueue = new LinkedBlockingQueue<>(queueSize);
        for (int i = 0; i < poolSize; i++) {
            WorkerThread worker = new WorkerThread();
            worker.start();
            threads.add(worker);
        }
    }
    
    public void execute(Runnable task) {
        if (shutdown) {
            throw new IllegalStateException("ThreadPool is shutdown");
        }
        try {
            workQueue.put(task);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    public void shutdown() {
        shutdown = true;
        for (WorkerThread worker : threads) {
            worker.interrupt();
        }
    }
    
    class WorkerThread extends Thread {
        @Override
        public void run() {
            while (!shutdown || !workQueue.isEmpty()) {
                try {
                    Runnable task = workQueue.take();
                    task.run();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
    }
}
```

5. **手写快速排序**
```java
public void quickSort(int[] arr, int low, int high) {
    if (low < high) {
        int pivot = partition(arr, low, high);
        quickSort(arr, low, pivot - 1);
        quickSort(arr, pivot + 1, high);
    }
}

private int partition(int[] arr, int low, int high) {
    int pivot = arr[high];
    int i = low - 1;
    for (int j = low; j < high; j++) {
        if (arr[j] < pivot) {
            i++;
            swap(arr, i, j);
        }
    }
    swap(arr, i + 1, high);
    return i + 1;
}

private void swap(int[] arr, int i, int j) {
    int temp = arr[i];
    arr[i] = arr[j];
    arr[j] = temp;
}
```

---

如果你需要我对内容进行精简版或加上更详细的示例答案，请告诉我你的偏好（例如"突出并发/JVM""附带详细解答"）。


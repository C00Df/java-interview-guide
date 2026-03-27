# Java 并发编程 & JVM 虚拟机 — 超级学习指南

> 📌 整理日期：2026-03-25
> 🎯 目标：面试突击 + 深入理解 + 实战应用
> 👤 余辰飞 · 4年Java后端 · 目标自研公司

---

## 📖 目录

- [第一章：并发编程基础](#第一章并发编程基础)
- [第二章：线程安全与锁机制](#第二章线程安全与锁机制)
- [第三章：JUC 核心工具类](#第三章juc-核心工具类)
- [第四章：线程池深度剖析](#第四章线程池深度剖析)
- [第五章：并发容器与数据结构](#第五章并发容器与数据结构)
- [第六章：并发设计模式与实战](#第六章并发设计模式与实战)
- [第七章：JVM 运行时数据区](#第七章jvm-运行时数据区)
- [第八章：垃圾回收机制](#第八章垃圾回收机制)
- [第九章：类加载机制](#第九章类加载机制)
- [第十章：JVM 调优与问题排查](#第十章jvm-调优与问题排查)
- [第十一章：高频面试题库](#第十一章高频面试题库)

---

# 第一章：并发编程基础

## 1.1 进程与线程的本质区别

```
┌─────────────────────────────────────────────────────────────┐
│                        进程 (Process)                       │
│  资源分配的基本单位                                          │
│  ├── 独立的内存空间（堆、栈、方法区）                          │
│  ├── 至少包含一个线程（主线程）                               │
│  └── 进程间通信需要 IPC 机制（Socket、管道、消息队列）          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        线程 (Thread)                        │
│  CPU 调度的基本单位                                          │
│  ├── 共享进程的资源（堆、方法区）                              │
│  ├── 拥有独立的栈空间和程序计数器                              │
│  ├── 线程间通信直接读写共享变量                               │
│  └── 上下文切换开销小（比进程快 10-100 倍）                   │
└─────────────────────────────────────────────────────────────┘
```

**核心理解**：
- 进程是"房子"，线程是房子里的"人"
- 一个进程可以有多个人（线程），共享客厅（堆）
- 每个人有自己的房间（栈）和笔记本（程序计数器）

## 1.2 线程的 4 种创建方式

### 方式一：继承 Thread 类

```java
// 第一步：继承 Thread
class MyThread extends Thread {
    
    // 第二步：重写 run() 方法（线程体）
    @Override
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println("线程1执行: " + i);
        }
    }
}

// 第三步：创建并启动
public class ThreadDemo {
    public static void main(String[] args) {
        MyThread t = new MyThread();
        t.start();  // 启动线程（不是 run()！）
        
        // main 线程继续执行
        for (int i = 0; i < 5; i++) {
            System.out.println("main线程执行: " + i);
        }
    }
}
```

**⚠️ 重要区分**：
- `start()`：启动新线程，线程处于就绪状态，等待 CPU 调度
- `run()`：普通方法调用，在当前线程执行，不会创建新线程

### 方式二：实现 Runnable 接口（推荐）

```java
// 第一步：实现 Runnable
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable 方式执行");
    }
}

// 第二步：创建 Thread 时传入 Runnable
public class RunnableDemo {
    public static void main(String[] args) {
        Thread t = new Thread(new MyRunnable());
        t.start();
    }
}
```

**为什么推荐 Runnable？**
- Java 单继承，Thread 继承了就没法继承其他类
- Runnable 更灵活，可以实现多个接口
- 便于线程池管理

### 方式三：实现 Callable 接口 + FutureTask（有返回值）

```java
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

// 第一步：实现 Callable（可以抛异常，有返回值）
class MyCallable implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        int sum = 0;
        for (int i = 1; i <= 100; i++) {
            sum += i;
        }
        return sum;  // 可以返回结果
    }
}

public class CallableDemo {
    public static void main(String[] args) throws Exception {
        // FutureTask 包装 Callable
        FutureTask<Integer> futureTask = new FutureTask<>(new MyCallable());
        
        // 传入 FutureTask 创建线程
        Thread t = new Thread(futureTask);
        t.start();
        
        // 获取线程执行结果（会阻塞等待）
        Integer result = futureTask.get();
        System.out.println("计算结果: " + result);  // 5050
    }
}
```

### 方式四：线程池（企业级推荐）

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadPoolDemo {
    public static void main(String[] args) {
        // 创建线程池
        ExecutorService executor = Executors.newFixedThreadPool(3);
        
        // 提交任务
        for (int i = 1; i <= 10; i++) {
            final int taskId = i;
            executor.execute(() -> {
                System.out.println("任务" + taskId + "由线程" + 
                    Thread.currentThread().getName() + "执行");
            });
        }
        
        // 关闭线程池
        executor.shutdown();
    }
}
```

**四种创建方式对比**：

| 方式 | 返回值 | 抛异常 | 资源共享 | 推荐场景 |
|------|--------|--------|----------|----------|
| Thread | ❌ | ❌ | 独立 | 需要继承（很少用） |
| Runnable | ❌ | ❌ | 共享 | 一般并发任务 |
| Callable | ✅ | ✅ | 共享 | 需要返回结果 |
| 线程池 | — | — | — | **企业级首选** |

## 1.3 线程的 6 种状态

```
         ┌─────────┐
         │  NEW    │ ← new Thread()，创建了对象但没启动
         └────┬────┘
              │ .start()
              ▼
    ┌─────────────────┐
    │    RUNNABLE     │ ← 可运行状态（就绪 + 正在运行）
    │ (Ready / Running)│   CPU调度到就Running，没调度到就Ready
    └────────┬────────┘
             │
      ┌──────┴──────┐
      │             │
      ▼             ▼
┌──────────┐   ┌──────────┐
│BLOCKED   │   │ WAITING  │
│ 阻塞     │   │ 无限等待  │
└────┬─────┘   └────┬─────┘
     │              │
     │ 获得锁       │ .wait() / .join()
     ▼              ▼
   就绪           就绪
      │              │
      │ notify()     │ notify() / notifyAll() / timeout
      ▼              ▼
```

**各状态详解**：

| 状态 | 含义 | 进入方式 |
|------|------|----------|
| **NEW** | 刚创建，未 start() | new Thread() |
| **RUNNABLE** | 可运行/运行中 | start()、wait后被唤醒、sleep结束 |
| **BLOCKED** | 等待锁，阻塞 | 进入 synchronized 代码块但锁被占用 |
| **WAITING** | 无限等待 | wait()、join()、LockSupport.park() |
| **TIMED_WAITING** | 限时等待 | sleep(ms)、wait(ms)、join(ms)、parkNanos() |
| **TERMINATED** | 执行完毕 | run()正常返回或抛出异常 |

## 1.4 线程常用方法详解

### sleep() vs wait() — 核心区别

| 对比项 | sleep() | wait() |
|--------|---------|--------|
| 所属 | Thread 类（静态方法） | Object 类 |
| 释放锁 | ❌ 不释放 | ✅ 释放 |
| 唤醒方式 | 时间到自动唤醒 | notify/notifyAll |
| 使用场景 | 暂停执行、轮询 | 线程间通信 |
| 位置 | 任意地方调用 | 必须在 synchronized 内 |

### join() — 等待线程执行完毕

```java
public class JoinDemo {
    public static void main(String[] args) throws InterruptedException {
        
        Thread t1 = new Thread(() -> {
            for (int i = 1; i <= 3; i++) {
                System.out.println("子线程: " + i);
                try { Thread.sleep(500); } catch (InterruptedException e) {}
            }
        });
        
        t1.start();
        
        // main 线程等待 t1 执行完毕
        t1.join();
        
        // 这行代码会在 t1 执行完毕后才会执行
        System.out.println("main 线程继续执行");
    }
}

// 输出顺序：
// 子线程: 1
// 子线程: 2
// 子线程: 3
// main 线程继续执行
```

### yield() vs sleep()

- `yield()`：只是提示，不保证真正让出，可能下次又抢到
- `sleep()`：保证休眠指定时间

### interrupt() — 中断线程

```java
public class InterruptDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                System.out.println("线程运行中...");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    // sleep/wait/join 会清除中断标志
                    // 建议在 catch 中重新设置中断标志
                    Thread.currentThread().interrupt();
                    break;
                }
            }
            System.out.println("线程退出");
        });
        
        t.start();
        Thread.sleep(3000);
        t.interrupt();  // 中断线程
    }
}
```

## 1.5 线程间的协作机制

### 生产者-消费者问题

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

class SharedData {
    private int data;           // 数据
    private boolean hasData;    // 是否有数据
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();   // 有数据
    private final Condition notFull = lock.newCondition();    // 没满
    
    private static final int MAX_SIZE = 10;
    
    // 生产者
    public void produce(int value) throws InterruptedException {
        lock.lock();
        try {
            while (hasData) {  // 有数据就等待
                notFull.await();
            }
            data = value;
            hasData = true;
            System.out.println("生产: " + value);
            notEmpty.signal();  // 通知消费者
        } finally {
            lock.unlock();
        }
    }
    
    // 消费者
    public int consume() throws InterruptedException {
        lock.lock();
        try {
            while (!hasData) {  // 没数据就等待
                notEmpty.await();
            }
            hasData = false;
            System.out.println("消费: " + data);
            notFull.signal();  // 通知生产者
            return data;
        } finally {
            lock.unlock();
        }
    }
}
```

### 使用 wait/notify 实现

```java
class Buffer {
    private int item;
    private boolean available = false;
    
    public synchronized void put(int item) throws InterruptedException {
        while (available) {
            wait();  // 缓冲区有数据，等待消费
        }
        this.item = item;
        available = true;
        System.out.println("生产: " + item);
        notify();  // 通知消费者
    }
    
    public synchronized int get() throws InterruptedException {
        while (!available) {
            wait();  // 缓冲区没有数据，等待生产
        }
        available = false;
        System.out.println("消费: " + item);
        notify();  // 通知生产者
        return item;
    }
}
```

**⚠️ 重要原则**：
- `wait()` 必须在 `while` 循环中，而不是 `if` 中
- 原因：多个线程被唤醒时，可能条件已经发生变化
- 使用 `notifyAll()` 而不是 `notify()`，除非确定只有一个等待线程

## 1.6 线程饥饿与死锁

### 死锁的四个必要条件

```
1. 互斥条件：资源只能被一个线程持有
2. 持有并等待：线程持有资源的同时请求其他资源
3. 不可抢占：资源不能被强制释放，只能主动释放
4. 循环等待：线程之间形成循环等待链
```

### 死锁代码演示

```java
public class DeadLockDemo {
    private static final Object LOCK_A = new Object();
    private static final Object LOCK_B = new Object();
    
    public static void main(String[] args) {
        // 线程1：先拿A，再拿B
        Thread t1 = new Thread(() -> {
            synchronized (LOCK_A) {
                System.out.println("线程1: 获得锁A");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                synchronized (LOCK_B) {
                    System.out.println("线程1: 获得锁B");
                }
            }
        });
        
        // 线程2：先拿B，再拿A（与线程1相反）
        Thread t2 = new Thread(() -> {
            synchronized (LOCK_B) {
                System.out.println("线程2: 获得锁B");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                synchronized (LOCK_A) {
                    System.out.println("线程2: 获得锁A");
                }
            }
        });
        
        t1.start();
        t2.start();
        // 两个线程相互等待对方持有的锁，形成死锁
    }
}
```

### 死锁的解决策略

```java
// 策略1：固定加锁顺序（解决死锁）
// 所有线程都以相同的顺序获取锁
public class FixedDeadLock {
    private static final Object LOCK_A = new Object();
    private static final Object LOCK_B = new Object();
    
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (LOCK_A) {  // 统一先A
                System.out.println("线程1: 获得锁A");
                synchronized (LOCK_B) {  // 再B
                    System.out.println("线程1: 获得锁B");
                }
            }
        });
        
        Thread t2 = new Thread(() -> {
            synchronized (LOCK_A) {  // 统一先A
                System.out.println("线程2: 获得锁A");
                synchronized (LOCK_B) {  // 再B
                    System.out.println("线程2: 获得锁B");
                }
            }
        });
        
        t1.start();
        t2.start();
        // 不会死锁，因为顺序一致
    }
}
```

### 检测死锁

```bash
# Linux/Mac 查看 Java 进程
jps -l

# 导出线程栈
jstack <pid>
```

---

# 第二章：线程安全与锁机制

## 2.1 什么是线程安全？

```java
// ❌ 非线程安全
class CounterUnsafe {
    private int count = 0;
    public void increment() {
        count++;  // 三个步骤：读→改→写，不是原子操作
    }
}

// ✅ 线程安全：使用 synchronized
class CounterSafe {
    private int count = 0;
    public synchronized void increment() {
        count++;
    }
}

// ✅ 线程安全：使用原子类
class CounterAtomic {
    private AtomicInteger count = new AtomicInteger(0);
    public void increment() {
        count.incrementAndGet();
    }
}
```

**线程不安全的原因**：
1. **可见性**：一个线程修改了变量，其他线程可能看不到
2. **原子性**：操作不是不可分割的
3. **有序性**：指令可能被重排序

## 2.2 synchronized 关键字深度解析

### synchronized 的三种用法

```java
public class SyncUsage {
    
    // 1. 修饰实例方法（锁对象 this）
    public synchronized void method1() {
        // 锁住当前对象，同一对象的所有 synchronized 方法互斥
    }
    
    // 2. 修饰静态方法（锁类对象）
    public static synchronized void method2() {
        // 锁住 Class 对象，所有对象共享这一把锁
    }
    
    // 3. 修饰代码块（锁指定对象）
    public void method3() {
        synchronized (this) {
            // 锁住当前对象
        }
        synchronized (SyncUsage.class) {
            // 锁住 Class，所有对象共享
        }
    }
}
```

### synchronized 锁的升级过程（JDK 1.6 优化）

```
┌──────────────┐
│    无锁      │  ← 对象刚创建
└──────┬───────┘
       │ 偏向锁启用
       ▼
┌──────────────┐
│   偏向锁     │  ← 只有一个线程访问同步块
│              │    Mark Word 记录线程ID
└──────┬───────┘
       │ 竞争发生
       ▼
┌──────────────┐
│  轻量级锁    │  ← 多线程但不同时竞争
│              │    CAS 自旋，Mark Word 替换
└──────┬───────┘
       │ 竞争激烈
       ▼
┌──────────────┐
│  重量级锁    │  ← 真正阻塞
│              │    OS mutex，不消耗 CPU
└──────────────┘
```

**各阶段详解**：

| 锁类型 | 原理 | 适用场景 | 优点 | 缺点 |
|--------|------|----------|------|------|
| **偏向锁** | 记录线程ID，后续直接进入 | 单线程访问同步块 | 无成本 | 多线程竞争时需要撤销 |
| **轻量级锁** | CAS 自旋替换 Mark Word | 多线程交替执行（不阻塞） | 不阻塞线程 | 自旋消耗 CPU |
| **重量级锁** | OS 互斥量，线程阻塞 | 竞争激烈 | 线程真正阻塞，不消耗CPU | 线程切换开销大 |

### synchronized 的可重入性

```java
public class ReentrantDemo {
    public synchronized void method1() {
        System.out.println("进入 method1");
        method2();  // 可以再次获取锁
    }
    
    public synchronized void method2() {
        System.out.println("进入 method2");
    }
    
    public static void main(String[] args) {
        ReentrantDemo demo = new ReentrantDemo();
        demo.method1();
    }
}
```

## 2.3 ReentrantLock 深入理解

### ReentrantLock vs synchronized

| 对比项 | synchronized | Lock |
|--------|--------------|------|
| 来源 | Java 关键字 | JDK 接口 |
| 获取/释放 | 自动（编译器负责） | 手动（在 finally 中释放） |
| 可中断 | ❌ | ✅（tryLock 可以超时中断） |
| 公平锁 | ❌（非公平） | ✅（可以设为公平锁） |
| 多条件 | ❌ | ✅（多个 Condition） |
| 性能 | JDK 1.6 后优化很多 | 高 |

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.TimeUnit;

public class LockUsage {
    private final ReentrantLock lock = new ReentrantLock();
    
    // 基础用法
    public void doSomething() {
        lock.lock();
        try {
            // 临界区
        } finally {
            lock.unlock();  // 必须释放
        }
    }
    
    // tryLock：尝试获取，不等待
    public void tryLockExample() {
        if (lock.tryLock()) {
            try {
                // 获取到了
            } finally {
                lock.unlock();
            }
        } else {
            // 没获取到，做其他事
        }
    }
    
    // tryLock 超时：等一段时间
    public boolean tryLockWithTimeout() throws InterruptedException {
        if (lock.tryLock(3, TimeUnit.SECONDS)) {
            try {
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }
    
    // 公平锁
    private final ReentrantLock fairLock = new ReentrantLock(true);  // true = 公平锁
}
```

### Condition 条件变量

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ConditionDemo {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notEmpty = lock.newCondition();
    private final Condition notFull = lock.newCondition();
    
    private final Object[] queue = new Object[10];
    private int count = 0;
    
    // 生产者
    public void put(Object obj) throws InterruptedException {
        lock.lock();
        try {
            while (count == queue.length) {
                notFull.await();  // 等待 notFull 信号
            }
            queue[count++] = obj;
            notEmpty.signal();  // 唤醒消费者
        } finally {
            lock.unlock();
        }
    }
    
    // 消费者
    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();  // 等待 notEmpty 信号
            }
            Object obj = queue[--count];
            notFull.signal();  // 唤醒生产者
            return obj;
        } finally {
            lock.unlock();
        }
    }
}
```

## 2.4 volatile 关键字深度剖析

### volatile 的两大特性

```java
public class VolatileDemo {
    // volatile 保证：可见性 + 有序性
    private volatile boolean flag = false;
    
    // 线程A
    public void writer() {
        flag = true;  // 写入主内存，禁止指令重排序
    }
    
    // 线程B
    public void reader() {
        while (!flag) {  // 每次都从主内存读取
            // 等待
        }
        // 一定能读到 true
    }
}
```

### volatile 不能保证原子性

```java
// ❌ volatile 不能解决 i++ 的线程安全问题
private volatile int i = 0;
public void increment() {
    i++;  // 不是原子操作！
}

// ✅ 解决方案1：synchronized
private int i = 0;
public synchronized void increment() { i++; }

// ✅ 解决方案2：AtomicInteger
private AtomicInteger i = new AtomicInteger(0);
public void increment() { i.incrementAndGet(); }
```

### volatile 的典型使用场景

```java
// 场景1：状态标志
public class StateFlag {
    private volatile boolean running = true;
    
    public void run() {
        while (running) {
            // 处理任务
        }
    }
    
    public void stop() {
        running = false;  // 其他线程立即看到变化
    }
}

// 场景2：单例模式双重检查锁
public class Singleton {
    private static volatile Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {  // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {  // 第二次检查
                    instance = new Singleton();
                    //  volatile 防止指令重排序
                    //  new Singleton() 分三步：
                    //  1. 分配内存
                    //  2. 调用构造方法
                    //  3. 赋值给引用
                    //  可能重排序为 1-3-2，导致其他线程拿到未初始化对象
                }
            }
        }
        return instance;
    }
}
```

## 2.5 CAS (Compare And Swap) 深度剖析

### CAS 原理

```
┌─────────────────────────────────────────────────┐
│                   CAS 原理                      │
│                                                 │
│  内存位置 V        预期值 A        新值 B        │
│      │                │               │         │
│      ▼                ▼               ▼         │
│  ┌─────────────────────────────────────────┐   │
│  │         Compare And Swap                │   │
│  │                                         │   │
│  │  if (V == A) {  // 比较                 │   │
│  │      V = B;      // 交换                │   │
│  │      return true;                       │   │
│  │  } else {                               │   │
│  │      return false;  // 不相等，重试     │   │
│  │  }                                      │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### CAS 的三大问题

| 问题 | 说明 | 解决方案 |
|------|------|----------|
| **ABA问题** | A→B→A，CAS认为没变 | AtomicStampedReference（带版本号） |
| **自旋开销** | 长时间循环CPU占用高 | 自适应自旋 |
| **只能保证单个变量** | 无法同时操作多个变量 | 使用 synchronized 或合并操作 |

### ABA 问题详解与解决

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicStampedReference;

// ❌ ABA 问题
public class ABAProblem {
    public static void main(String[] args) {
        AtomicInteger ref = new AtomicInteger(100);
        
        Thread t1 = new Thread(() -> {
            boolean success = ref.compareAndSet(100, 101);  // A→B
            System.out.println("t1: " + success);
        });
        
        Thread t2 = new Thread(() -> {
            ref.compareAndSet(100, 101);   // A→B
            ref.compareAndSet(101, 100);   // B→A（变回来了！）
            System.out.println("t2: 100→101→100");
        });
        
        Thread t3 = new Thread(() -> {
            // t3 不知道中间被改过！
            boolean success = ref.compareAndSet(100, 200);  // 成功！
            System.out.println("t3: " + success + " (值为: " + ref.get() + ")");
        });
        
        t2.start();
        t1.start();
        t3.start();
    }
}

// ✅ 解决方案：AtomicStampedReference
public class ABASolution {
    public static void main(String[] args) {
        AtomicStampedReference<Integer> ref = 
            new AtomicStampedReference<>(100, 0);
        
        Thread t1 = new Thread(() -> {
            int[] stamp = new int[1];
            Integer value = ref.get(stamp);
            System.out.println("t1 读取: " + value + ", stamp: " + stamp[0]);
        });
        
        // 版本号递增，解决 ABA 问题
        ref.compareAndSet(100, 101, 0, 1);  // stamp 从 0 变成 1
        ref.compareAndSet(101, 100, 1, 2);  // stamp 从 1 变成 2
        
        t1.start();
    }
}
```

### Atomic 原子类家族

```java
import java.util.concurrent.atomic.*;

// 基础类型
AtomicInteger        // 原子整数
AtomicLong           // 原子长整型
AtomicBoolean        // 原子布尔

// 引用类型
AtomicReference      // 原子引用
AtomicStampedReference  // 带版本号的引用
AtomicMarkableReference  // 带标记的引用

// 数组
AtomicIntegerArray    // 原子整数数组
AtomicLongArray       // 原子长整型数组
AtomicReferenceArray  // 原子引用数组

// 字段更新器
AtomicIntegerFieldUpdater   // 更新对象的 int 字段
AtomicLongFieldUpdater      // 更新对象的 long 字段
AtomicReferenceFieldUpdater // 更新对象的引用字段

// 高并发计数器
LongAdder   // 分段累加，比 AtomicLong 性能更好
LongAccumulator  // 通用累加器

// 使用示例
public class AtomicDemo {
    public static void main(String[] args) {
        // AtomicInteger 基本操作
        AtomicInteger i = new AtomicInteger(0);
        i.get();           // 获取值
        i.set(10);         // 设置值
        i.incrementAndGet();  // i++
        i.decrementAndGet();  // i--
        i.addAndGet(5);      // i += 5
        i.compareAndSet(10, 20);  // CAS
        
        // LongAdder：高并发计数（分段锁）
        LongAdder counter = new LongAdder();
        counter.increment();  // 高并发下比 AtomicLong 快
        counter.sum();
    }
}
```

---

# 第三章：JUC 核心工具类

## 3.1 CountDownLatch（倒计时门闩）

### 核心概念
- 倒数计数器，初始值为 N
- 每调用一次 `countDown()`，计数器减 1
- `await()` 阻塞，直到计数器为 0
- **一次性**：计数到 0 后不能重置

### 典型场景：主线程等待多个子任务完成

```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        int taskCount = 3;
        CountDownLatch latch = new CountDownLatch(taskCount);
        
        // 启动3个子任务
        for (int i = 1; i <= taskCount; i++) {
            final int taskId = i;
            new Thread(() -> {
                try {
                    System.out.println("任务" + taskId + "开始执行");
                    Thread.sleep((long) (Math.random() * 1000));
                    System.out.println("任务" + taskId + "执行完成");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown();  // 计数减1
                }
            }).start();
        }
        
        // 主线程等待所有任务完成
        System.out.println("主线程: 等待任务完成...");
        latch.await();  // 阻塞，直到计数为0
        System.out.println("所有任务完成，主线程继续执行");
    }
}
```

### 常见面试题：10个线程同时执行，如何保证主线程等所有子线程执行完再执行？

```java
// 方法1：CountDownLatch
public class Solution1 {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(10);
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                try {
                    // 执行任务
                    System.out.println("执行完毕");
                } finally {
                    latch.countDown();
                }
            }).start();
        }
        latch.await();
        System.out.println("主线程执行");
    }
}

// 方法2：CompletableFuture
public class Solution2 {
    public static void main(String[] args) throws Exception {
        CompletableFuture<?>[] futures = new CompletableFuture[10];
        for (int i = 0; i < 10;i++) {
            futures[i] = CompletableFuture.runAsync(() -> {
                System.out.println("执行完毕");
            });
        }
        CompletableFuture.allOf(futures).join();
        System.out.println("主线程执行");
    }
}
```

## 3.2 CyclicBarrier（循环栅栏）

### 核心概念
- 循环栅栏，初始值为 N
- 每调用一次 `await()`，计数减 1
- 当计数减到 0 时，所有等待线程同时继续执行
- **可循环使用**：计数器重置，可以再次使用

### 典型场景：多线程数据汇总

```java
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CyclicBarrierDemo {
    public static void main(String[] args) {
        int threadCount = 5;
        CyclicBarrier barrier = new CyclicBarrier(threadCount, () -> {
            // 所有线程到达后执行的汇总任务
            System.out.println("===== 所有数据准备完成，开始汇总 =====");
        });
        
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        
        for (int i = 1; i <= threadCount; i++) {
            final int id = i;
            executor.submit(() -> {
                try {
                    // 模拟各自处理数据
                    String data = "线程" + id + "的数据";
                    System.out.println("线程" + id + "准备好了: " + data);
                    
                    // 等待其他线程
                    barrier.await();
                    
                    // 汇总后各自继续执行
                    System.out.println("线程" + id + "开始后续处理");
                    
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
        }
        
        executor.shutdown();
    }
}
```

### CountDownLatch vs CyclicBarrier

| 对比项 | CountDownLatch | CyclicBarrier |
|--------|---------------|---------------|
| 用途 | 一个线程等待N个线程完成 | N个线程互相等待 |
| 可重置 | ❌ 一次性 | ✅ 可循环使用 |
| 计数器 | 只减不增 | 减到0后自动重置 |
| 典型场景 | 主线程等待子任务 | 多阶段任务，每个阶段汇总 |

## 3.3 Semaphore（信号量）

### 核心概念
- 控制同时访问资源的线程数量
- `acquire()`：获取许可，许可不足则阻塞
- `release()`：释放许可
- 可以用于**限流**

### 典型场景：连接池限流

```java
import java.util.concurrent.Semaphore;

public class SemaphoreDemo {
    public static void main(String[] args) {
        // 模拟数据库连接池，只有3个连接
        Semaphore semaphore = new Semaphore(3);
        
        for (int i = 1; i <= 10; i++) {
            final int id = i;
            new Thread(() -> {
                try {
                    // 获取连接
                    semaphore.acquire();
                    System.out.println("线程" + id + "获得连接");
                    
                    // 模拟使用连接
                    Thread.sleep((long) (Math.random() * 2000));
                    
                    // 释放连接
                    semaphore.release();
                    System.out.println("线程" + id + "释放连接");
                    
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
    }
}
```

### Semaphore 进阶用法

```java
import java.util.concurrent.Semaphore;

public class SemaphoreAdvance {
    public static void main(String[] args) {
        // true = 公平模式（按等待顺序获取）
        Semaphore fairSemaphore = new Semaphore(3, true);
        
        // 尝试获取（不阻塞）
        if (fairSemaphore.tryAcquire()) {
            try {
                // 获取到了
            } finally {
                fairSemaphore.release();
            }
        } else {
            // 没获取到
        }
        
        // 带超时的获取
        try {
            fairSemaphore.tryAcquire(3, java.util.concurrent.TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 3.4 Exchanger（数据交换器）

### 核心概念
- 两个线程之间交换数据
- 一方先到达会阻塞，等待另一方

```java
import java.util.concurrent.Exchanger;

public class ExchangerDemo {
    public static void main(String[] args) {
        Exchanger<String> exchanger = new Exchanger<>();
        
        Thread t1 = new Thread(() -> {
            try {
                String data = "A的数据";
                System.out.println("线程A: 携带数据: " + data);
                
                // 交换数据，阻塞等待另一方
                String received = exchanger.exchange(data);
                System.out.println("线程A: 收到数据: " + received);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "线程A");
        
        Thread t2 = new Thread(() -> {
            try {
                String data = "B的数据";
                Thread.sleep(1000);  // 稍晚一点
                System.out.println("线程B: 携带数据: " + data);
                
                String received = exchanger.exchange(data);
                System.out.println("线程B: 收到数据: " + received);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "线程B");
        
        t1.start();
        t2.start();
    }
}
```

## 3.5 Phaser（阶段同步器）

### 核心概念
- 比 CyclicBarrier 更灵活
- 可以动态注册参与者
- 支持多个阶段的同步

```java
import java.util.concurrent.Phaser;

public class PhaserDemo {
    public static void main(String[] args) {
        Phaser phaser = new Phaser(3);  // 3个参与者
        
        for (int i = 1; i <= 3; i++) {
            final int id = i;
            new Thread(() -> {
                // 阶段1
                System.out.println("线程" + id + " 完成阶段1");
                phaser.arriveAndAwaitAdvance();  // 等待其他线程
                
                // 阶段2
                System.out.println("线程" + id + " 开始阶段2");
                phaser.arriveAndAwaitAdvance();
                
                System.out.println("线程" + id + " 完成所有阶段");
            }).start();
        }
    }
}
```

---

# 第四章：线程池深度剖析
## 4.1 为什么需要线程池？

**不用线程池的问题：**
`java
// ❌ 每次请求都创建新线程
public class BadPractice {
    public static void main(String[] args) {
        while (true) {
            new Thread(() -> {
                // 处理请求
            }).start();
        }
        // 问题：
        // 1. 创建/销毁线程开销大
        // 2. 线程数量不可控
        // 3. 资源耗尽
    }
}
`

**线程池的优势：**
- 复用线程，减少创建/销毁开销
- 控制并发数量，避免资源耗尽
- 提供管理功能：拒绝策略、监控
- 提高响应速度

## 4.2 线程池的创建方式

### 方式一：Executors 工具类（不推荐，有隐患）

`java
import java.util.concurrent.Executors;

public class BadExecutors {
    public static void main(String[] args) {
        // ❌ FixedThreadPool：请求队列无限大，可能 OOM
        ExecutorService pool1 = Executors.newFixedThreadPool(10);
        // 实际：LinkedBlockingQueue 无界队列
        
        // ❌ CachedThreadPool：最大线程数无限，任务多会创建大量线程
        ExecutorService pool2 = Executors.newCachedThreadPool();
        // 实际：maximumPoolSize = Integer.MAX_VALUE
        
        // ❌ SingleThreadExecutor：请求队列无限大
        ExecutorService pool3 = Executors.newSingleThreadExecutor();
        
        // ⚠️ ScheduledExecutorService：定时任务
        ScheduledExecutorService pool4 = Executors.newScheduledThreadPool(3);
    }
}
`

### 方式二：ThreadPoolExecutor（推荐）

`java
import java.util.concurrent.*;

public class GoodExecutor {
    public static void main(String[] args) {
        int corePoolSize = 5;           // 核心线程数
        int maxPoolSize = 10;            // 最大线程数
        long keepAliveTime = 60L;       // 非核心线程存活时间
        TimeUnit unit = TimeUnit.SECONDS;
        BlockingQueue<Runnable> workQueue = new LinkedBlockingQueue<>(100);  // 有界队列
        ThreadFactory threadFactory = Executors.defaultThreadFactory();
        RejectedExecutionHandler handler = new ThreadPoolExecutor.AbortPolicy();
        
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            corePoolSize,
            maxPoolSize,
            keepAliveTime,
            unit,
            workQueue,
            threadFactory,
            handler
        );
        
        for (int i = 0; i < 20; i++) {
            executor.execute(() -> System.out.println("任务执行"));
        }
        
        executor.shutdown();
    }
}
```

## 4.3 线程池的 7 大参数详解

| 参数 | 说明 | 常见设置 |
|------|------|----------|
| **corePoolSize** | 核心线程数，即使空闲也保留 | CPU密集型：核心数+1；IO密集型：核心数*2 |
| **maximumPoolSize** | 最大线程数 | corePoolSize ~ 2*corePoolSize |
| **keepAliveTime** | 非核心线程空闲存活时间 | 60秒 |
| **unit** | 时间单位 | TimeUnit.SECONDS |
| **workQueue** | 任务等待队列 | LinkedBlockingQueue（有界） |
| **threadFactory** | 创建线程的工厂 | defaultThreadFactory() |
| **handler** | 拒绝策略 | AbortPolicy |

## 4.4 线程池执行流程

```
提交任务
    ↓
核心线程数 < corePoolSize ?
    ↓ 是                    ↓ 否
创建新线程              任务队列未满 ?
    ↓                       ↓ 是              ↓ 否
执行任务             加入队列等待      最大线程数 < maxPoolSize ?
                                               ↓ 是              ↓ 否
                                           创建非核心线程    执行拒绝策略
```

**执行流程口诀：**
1. 先看核心线程满了没
2. 核心满了看队列满了没
3. 队列满了看最大线程满了没
4. 最大线程满了执行拒绝策略

## 4.5 4 种拒绝策略

| 策略 | 行为 | 使用场景 |
|------|------|----------|
| **AbortPolicy** | 抛 RejectedExecutionException（默认） | 需要显式感知拒绝 |
| **DiscardPolicy** | 静默丢弃新任务 | 任务不重要，可以丢失 |
| **DiscardOldestPolicy** | 丢弃队列最老的任务，重试 | 优先级任务 |
| **CallerRunsPolicy** | 由提交任务的线程执行 | 限流 + 缓冲 |

## 4.6 线程池大小设置

`java
// CPU 密集型：计算密集，CPU 被充分利用
// 公式：核心数 + 1
int cpuCores = Runtime.getRuntime().availableProcessors();
int poolSize = cpuCores + 1;

// IO 密集型：大量等待 IO
// 公式：核心数 / (1 - 阻塞系数)  或  核心数 * 2
int ioCores = Runtime.getRuntime().availableProcessors();
int ioPoolSize = ioCores * 2;
`

## 4.7 线程池状态

`java
// 5种状态：
// RUNNING:  接受新任务，处理队列任务
// SHUTDOWN: 不接受新任务，但处理队列任务
// STOP:     不接受新任务，不处理队列任务，中断正在执行的任务
// TIDYING:  所有任务终止，workerCount=0，准备调用 terminated()
// TERMINATED: terminated() 执行完毕
`

## 4.8 常见线程池类型选择

| 类型 | 特点 | 适用场景 |
|------|------|----------|
| **FixedThreadPool** | 固定线程数，无界队列 | 负载稳定的任务 |
| **CachedThreadPool** | 线程数无限 | 短期异步任务 |
| **SingleThreadPool** | 单线程 | 保证顺序执行 |
| **ScheduledPool** | 定时/周期性任务 | 定时任务 |
| **WorkStealingPool** | ForkJoinPool | 计算密集型 |

## 4.9 关闭线程池

`java
// 方法1：不再接受新任务，等已提交的任务执行完
executor.shutdown();
boolean shutdown = executor.awaitTermination(60, TimeUnit.SECONDS);

// 方法2：立即停止所有任务
List<Runnable> tasks = executor.shutdownNow();
`

---

# 第五章：并发容器与数据结构

## 5.1 并发容器概览

`
Java 并发容器体系
├── ConcurrentHashMap    → ConcurrentSkipListMap
├── CopyOnWriteArrayList → ConcurrentLinkedQueue
├── BlockingQueue（阻塞队列家族）
│   ├── ArrayBlockingQueue
│   ├── LinkedBlockingQueue
│   ├── SynchronousQueue
│   ├── PriorityBlockingQueue
│   ├── DelayQueue
│   └── LinkedTransferQueue
└── ThreadLocal → InheritableThreadLocal
`

## 5.2 ConcurrentHashMap vs Hashtable

| 对比项 | Hashtable | ConcurrentHashMap |
|--------|-----------|-------------------|
| 实现 | 早期遗留 | 分段锁 → CAS+synchronized |
| 锁粒度 | 全表锁 | 桶级别 |
| 并发度 | 低 | 高 |
| JDK1.7 | synchronized | Segment 分段锁 |
| JDK1.8 | synchronized | CAS + synchronized |

### ConcurrentHashMap 原理（JDK 1.8）

`java
// put 流程：
// 1. 计算哈希，确定桶位置
// 2. 如果桶为空，使用 CAS 添加
// 3. 如果桶有元素，使用 synchronized 锁定并添加
// 4. 如果链表过长（>8），转为红黑树
`

### ConcurrentHashMap 常用操作

`java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// 基本操作
map.put("a", 1);
map.get("a");
map.remove("a");
map.size();

// 原子操作（JUC 特色）
map.putIfAbsent("a", 1);   // 不存在才添加
map.replace("a", 1, 2);    // 替换（oldValue匹配才替换）
map.computeIfAbsent("key", k -> 1);  // 不存在则计算并添加
map.merge("key", 1, Integer::sum);    // 合并
`

## 5.3 CopyOnWriteArrayList

### 原理：读写分离，写时复制

`java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

// 添加元素（每次复制整个数组，开销大）
list.add("a");

// 读取（无锁，直接读）
String first = list.get(0);

// 迭代器（快照，弱一致性）
for (String item : list) {
    // 迭代器创建时的快照
}
`

### 使用场景

| 适用 | 不适用 |
|------|--------|
| 读多写少（读 >> 写） | 写多读少 |
| 并发迭代，不修改 | 需要严格一致性 |

## 5.4 BlockingQueue（阻塞队列）

### 核心接口

`java
public interface BlockingQueue<E> extends Queue<E> {
    // 阻塞
    void put(E e) throws InterruptedException;  // 队列满则阻塞
    E take() throws InterruptedException;       // 队列空则阻塞
    
    // 超时
    boolean offer(E e, long timeout, TimeUnit unit);
    E poll(long timeout, TimeUnit unit);
}
`

### 阻塞队列家族

| 队列 | 特点 | 适用场景 |
|------|------|----------|
| **ArrayBlockingQueue** | 有界数组，FIFO | 固定容量限流 |
| **LinkedBlockingQueue** | 可选有界/无界链表 | 常见选择 |
| **SynchronousQueue** | 不存储元素，直接传递 | 零库存交接 |
| **PriorityBlockingQueue** | 优先级排序，无界 | 优先级任务 |
| **DelayQueue** | 延迟获取，无界 | 定时任务调度 |

## 5.5 ThreadLocal 深度剖析

### 基本使用

`java
private static ThreadLocal<SimpleDateFormat> dateFormat = 
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
`

### 内存泄漏问题

`
ThreadLocalMap 的 Entry 结构：
- key（ThreadLocal）使用弱引用
- value 使用强引用

问题：
1. ThreadLocal 被 GC 回收 → key = null
2. value 仍被 Entry 强引用
3. 线程存活 → value 不会被回收
4. 线程池复用 → 内存持续泄漏

解决方案：手动 remove()
`

`java
// ✅ 正确用法：在 finally 中清理
public class SafeThreadLocal {
    private static ThreadLocal<String> tl = new ThreadLocal<>();
    
    public static String get() {
        try {
            return tl.get();
        } finally {
            tl.remove();  // 每次用完清理
        }
    }
}
`

---

# 第六章：并发设计模式与实战

## 6.1 线程安全的单例模式

### 方案1：双重检查锁 + volatile

`java
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                    // volatile 防止指令重排序
                }
            }
        }
        return instance;
    }
}
`

### 方案2：静态内部类（推荐）

`java
public class Singleton {
    private Singleton() {}
    
    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
`

### 方案3：枚举（最安全）

`java
public enum Singleton {
    INSTANCE;
    public void doSomething() {}
}
`

## 6.2 CompletableFuture 实战

`java
import java.util.concurrent.*;

CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> "Hello")
    .thenApply(s -> s + " World")
    .thenApply(String::toUpperCase)
    .exceptionally(ex -> "Error");

String result = future.get();
`

### 常用操作

`java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "B");

// 组合
f1.thenCombine(f2, (a, b) -> a + b);  // 两个都完成后组合
f1.thenCompose(a -> ...);              // 扁平化链式调用

// 等待
CompletableFuture.allOf(f1, f2).join();   // 等待所有完成
CompletableFuture.anyOf(f1, f2).join();  // 任一完成
`

## 6.3 限流模式

### 信号量限流

`java
import java.util.concurrent.Semaphore;

Semaphore SEMAPHORE = new Semaphore(10);

public void handle(int requestId) {
    try {
        if (SEMAPHORE.tryAcquire(1, 100, TimeUnit.MILLISECONDS)) {
            try {
                // 处理请求
            } finally {
                SEMAPHORE.release();
            }
        } else {
            // 被限流
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
`

---

# 第七章：JVM 运行时数据区

## 7.1 JVM 内存结构全景图

`
┌────────────────────────────────────────────────────────────────────┐
│                        JVM 运行时数据区                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                        堆 (Heap)                            │  │
│  │                    【线程共享】                               │  │
│  │  ┌─────────────────┐          ┌────────────────────────────┐ │  │
│  │  │    新生代        │          │        老年代              │ │  │
│  │  │  ┌─────┬──────┐ │          │                            │ │  │
│  │  │  │Eden │ S0 │S1│ │          │    长期存活的对象          │ │  │
│  │  │  └─────┴──────┘ │          │                            │ │  │
│  │  └─────────────────┘          └────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    虚拟机栈 (VM Stack)                        │  │
│  │                    【线程私有】                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                 方法区 (Method Area) / 元空间                  │  │
│  │                    【线程共享】                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
`

## 7.2 各区域详解

### 堆（Heap）

| 区域 | 大小比例 | 说明 |
|------|----------|------|
| 新生代 (Young) | 堆的 1/3 | Eden + Survivor0 + Survivor1 |
| 老年代 (Old) | 堆的 2/3 | 长期存活的对象 |
| Eden | 新生代的 8/10 | 新对象优先分配 |
| Survivor | 新生代的 1/10 × 2 | 存放 Minor GC 后存活的对象 |

### 虚拟机栈

**栈帧结构：**
`
┌─────────────────────┐
│   局部变量表        │ ← 存放方法参数和局部变量
├─────────────────────┤
│   操作数栈          │ ← 指令操作的数据存放
├─────────────────────┤
│   动态链接          │ ← 指向运行时常量池的引用
├─────────────────────┤
│   方法返回地址      │ ← 方法返回后继续执行的位置
└─────────────────────┘
`

### 常见错误

| 错误 | 原因 | 场景 |
|------|------|------|
| **StackOverflowError** | 栈深度过大 | 递归调用没有终止条件 |
| **OutOfMemoryError** | 栈扩展失败 | 创建大量线程 |

### 方法区 / 元空间

| 版本 | 实现 | 位置 | 大小 |
|------|------|------|------|
| JDK 1.7 | 永久代 (PermGen) | JVM 内存 | 受限 |
| JDK 1.8+ | 元空间 (Metaspace) | 本地内存 | 可动态扩展 |

---

# 第八章：垃圾回收机制

## 8.1 如何判断对象已"死"？

### 引用计数法

- 每个对象有个引用计数器
- 为0时回收
- **问题：** 无法处理循环引用

### 可达性分析（GC Roots）

`java
// GC Roots 包括：
// 1. 虚拟机栈（栈帧中的本地变量表）中引用的对象
// 2. 方法区中类静态属性引用的对象
// 3. 方法区中常量引用的对象
// 4. 本地方法栈中 JNI 引用的对象
`

## 8.2 四种引用类型

| 类型 | 回收时机 | 典型应用 |
|------|----------|----------|
| **强引用** | 永远不会回收 | Object obj = new Object() |
| **软引用** | 内存不足时回收 | 缓存 |
| **弱引用** | 下次GC时回收 | ThreadLocal、WeakHashMap |
| **虚引用** | 随时可回收 | 跟踪对象回收时机 |

`java
import java.lang.ref.*;

// 软引用 - 内存不足时回收
SoftReference<byte[]> softRef = new SoftReference<>(new byte[1024 * 1024 * 10]);

// 弱引用 - 下次GC时回收
WeakReference<byte[]> weakRef = new WeakReference<>(new byte[1024]);

// 虚引用 - 用于跟踪
PhantomReference<byte[]> phantomRef = new PhantomReference<>(new byte[1024], null);
`

## 8.3 垃圾回收算法

### 标记-清除算法

`
步骤：
1. 标记所有存活对象
2. 清除所有未标记对象

缺点：产生内存碎片
`

### 复制算法

`
原理：将内存分为两块，每次只用一块
      存活对象复制到另一块，然后清除整块

优点：没有内存碎片
缺点：内存利用率只有50%

应用：新生代（对象存活率低）
`

### 标记-整理算法

`
步骤：
1. 标记存活对象
2. 整理（移动）存活对象到一端
3. 清除边界外的对象

优点：无碎片
缺点：移动开销

应用：老年代
`

## 8.4 分代收集策略

`
┌─────────────────────────────────────────────────────────────┐
│                      分代收集                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   新生代 (Minor GC)           老年代 (Full GC)            │
│   ┌─────────────────────┐     ┌─────────────────────┐     │
│   │ Eden │ S0  │  S1   │ ──→ │                     │     │
│   │  8/10 │ 1/10│  1/10 │     │    老年代空间       │     │
│   └───────┴─────┴───────┘     └─────────────────────┘     │
│        ↑ 复制算法                    ↑ 标记-整理算法        │
│                                                             │
└─────────────────────────────────────────────────────────────┘

对象晋升老年代的条件：
1. 大对象直接进入（超过阈值）
2. 长期存活（默认15岁）
3. Survivor区相同年龄对象超过50%
`

## 8.5 垃圾收集器

### 新生代收集器

| 收集器 | 特点 | 算法 |
|--------|------|------|
| **Serial** | 单线程，STW | 复制 |
| **ParNew** | Serial多线程版 | 复制 |
| **Parallel Scavenge** | 吞吐量优先 | 复制 |

### 老年代收集器

| 收集器 | 特点 | 算法 |
|--------|------|------|
| **Serial Old** | 单线程，STW | 标记-整理 |
| **Parallel Old** | 多线程，吞吐量优先 | 标记-整理 |
| **CMS** | 并发低停顿 | 标记-清除 |

### 全堆收集器

| 收集器 | 特点 |
|--------|------|
| **G1** | 分区收集，可预测停顿 |
| **ZGC** | 低延迟，TB级堆，支持着色指针 |
| **Shenandoah** | 低延迟，不需要GC Roots |

### CMS 收集器

`
收集过程：
1. 初始标记 (STW) - 标记 GC Roots 直接引用的对象
2. 并发标记 - 从 GC Roots 出发标记存活对象
3. 重新标记 (STW) - 修正并发标记期间变动的对象
4. 并发清除 - 清除未标记的对象

优点：并发收集，低停顿
缺点：
- CPU敏感
- 无法处理浮动垃圾（Concurrent Mode Failure）
- 内存碎片
`

### G1 收集器

`
特点：
- 整堆收集器，不再区分新生代/老年代
- 将堆划分为多个 Region
- 优先回收垃圾最多的 Region
- 可预测停顿时间（-XX:MaxGCPauseMillis）

过程：初始标记 → 并发标记 → 最终标记 → 筛选回收
`

### ZGC

`
特点：
- 低延迟（停顿 < 1ms）
- 支持 TB 级堆
- 并发收集
- 使用着色指针和读屏障

缺点：
- 吞吐量略低
- 需要更多内存
`

---

# 第九章：类加载机制

## 9.1 类加载过程

`
加载 → 验证 → 准备 → 解析 → 初始化
`

| 阶段 | 工作内容 |
|------|----------|
| **加载** | 读取.class文件，生成Class对象 |
| **验证** | 文件格式、元数据、字节码、符号引用验证 |
| **准备** | 为类变量分配内存，设置默认值 |
| **解析** | 符号引用 → 直接引用 |
| **初始化** | 执行<clinit>()方法（静态变量赋值+静态代码块） |

## 9.2 类加载器

`
        Bootstrap ClassLoader (启动类加载器)
                    ↑
        Extension ClassLoader (扩展类加载器)
                    ↑
        Application ClassLoader (应用程序类加载器)
                    ↑
        自定义 ClassLoader
`

| 加载器 | 负责 | 路径 |
|--------|------|------|
| Bootstrap | 加载核心类库 | jre/lib |
| Extension | 加载扩展类 | jre/lib/ext |
| Application | 加载classpath | 用户代码 |
| Custom | 用户自定义 | 自定义 |

## 9.3 双亲委派模型

### 工作流程

`
类加载请求
     ↓
检查是否已加载
     ↓ 是 → 返回
     ↓ 否
委派给父加载器
     ↓
父加载器检查是否已加载
     ↓ 是 → 返回
     ↓ 否
     ...（递归向上）
     ↓
Bootstrap ClassLoader
     ↓
自己尝试加载
     ↓
找不到则向下传
`

### 优点

1. **避免类重复加载**：父加载器加载过的类，子加载器不会再次加载
2. **保证核心类安全**：防止自定义java.lang.String覆盖核心类

### 打破双亲委派

| 场景 | 原因 |
|------|------|
| Tomcat | 每个Web应用独立加载类 |
| SPI机制 | JDBC等，父加载器请求子加载器加载 |
| 热加载 | OSGI模块化 |

---

# 第十章：JVM 调优与问题排查

## 10.1 常用 JVM 参数

`ash
# 堆内存
-Xms512m              # 初始堆大小
-Xmx2g                # 最大堆大小
-Xmn256m              # 新生代大小

# 元空间
-XX:MetaspaceSize=128m
-XX:MaxMetaspaceSize=256m

# 垃圾收集器
-XX:+UseG1GC          # 使用G1收集器
-XX:+UseParallelGC    # 使用Parallel GC
-XX:+UseConcMarkSweepGC  # 使用CMS

# GC日志
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:gc.log

# OOM时生成dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/to/dump

# 停顿时间
-XX:MaxGCPauseMillis=200
`

## 10.2 调优目标与策略

| 目标 | 策略 | 收集器选择 |
|------|------|------------|
| 低延迟 | 减少STW时间 | CMS、G1、ZGC |
| 高吞吐 | 最大化CPU利用率 | Parallel GC |
| 大内存 | 避免Full GC | G1、ZGC |

## 10.3 问题排查工具

| 工具 | 用途 |
|------|------|
| **jps** | 查看Java进程 |
| **jstat** | 监控GC、类加载、编译 |
| **jmap** | 生成堆dump |
| **jstack** | 生成线程dump |
| **jvisualvm** | 可视化监控 |
| **MAT** | 内存分析工具 |
| **Arthas** | 阿里巴巴开源诊断工具 |

### 常用命令

`ash
# 查看Java进程
jps -l

# 查看GC情况
jstat -gcutil <pid> 1000

# 生成堆dump
jmap -dump:format=b,file=heap.hprof <pid>

# 查看线程栈
jstack <pid>

# Arthas 常用命令
# 启动
java -jar arthas-boot.jar

# 查看JVM信息
dashboard

# 反编译类
jad com.example.MyClass

# 方法监控
watch com.example.MyClass methodName '{params,returnObj}'
`

---

# 第十一章：高频面试题库

## JUC 并发编程面试题

### Q1: synchronized 和 ReentrantLock 的区别？

**答：**
1. 实现层面：synchronized是JVM关键字，Lock是API
2. 释放锁：synchronized自动释放，Lock需手动unlock
3. 功能：Lock支持可中断、可超时、公平锁、多条件变量
4. 性能：JDK1.6后synchronized优化，性能接近

### Q2: volatile 能保证线程安全吗？

**答：** 不能保证原子性，只能保证可见性和有序性。
i++操作分三步：读-改-写，volatile无法保证这三步原子性。

### Q3: 线程池的核心参数？

**答：** 7个参数：corePoolSize、maximumPoolSize、keepAliveTime、unit、workQueue、threadFactory、handler

### Q4: CountDownLatch 和 CyclicBarrier 的区别？

**答：**
- CountDownLatch：一个/多个线程等待其他线程完成，一次性
- CyclicBarrier：多个线程互相等待，到达屏障后一起执行，可循环

### Q5: 什么是 ABA 问题？如何解决？

**答：** CAS检查时值从A→B→A，CAS认为没变。使用AtomicStampedReference加版本号解决。

### Q6: 什么是自旋锁？有什么问题？

**答：** 获取锁失败时，循环重试而不是阻塞。问题：长时间自旋消耗CPU。解决方案：自适应自旋（由JVM动态调整自旋次数）。

### Q7: ThreadLocal 内存泄漏是怎么发生的？

**答：** ThreadLocalMap的key是弱引用，value是强引用。当ThreadLocal被GC后，key=null，但value仍被持有。线程池场景下线程复用，value无法释放。解决：手动remove()。

### Q8: ConcurrentHashMap 在 JDK 1.8 有什么变化？

**答：** JDK1.7用分段锁，JDK1.8改为CAS+synchronized。synchronized只锁定当前链表或红黑树头节点，并发度更高。

---

## JVM 面试题

### Q1: JVM 内存分区？

**答：** 堆、虚拟机栈、本地方法栈、程序计数器、方法区（元空间）

### Q2: 什么情况下对象会进入老年代？

**答：**
1. 大对象（超过阈值）
2. 长期存活（默认15岁）
3. Survivor区相同年龄对象超过50%

### Q3: 垃圾回收算法有哪些？

**答：** 标记-清除、标记-整理、复制算法。新生代用复制，老年代用标记-清除/整理。

### Q4: CMS 和 G1 的区别？

**答：**
- CMS：并发低停顿，标记-清除，有碎片，无法处理浮动垃圾
- G1：整堆收集，分区管理，可预测停顿，无碎片

### Q5: 类加载的双亲委派模型？

**答：** 加载请求先委托父加载器，父加载器能加载就加载，不能则子加载器加载。优点：避免重复加载，保证核心类安全。

### Q6: Full GC 触发条件？

**答：**
1. 老年代空间不足
2. 方法区空间不足
3. System.gc()调用
4. Minor GC后晋升对象大小超过老年代剩余空间

### Q7: 什么是 OOM？如何排查？

**答：** OutOfMemoryError，内存溢出。排查：1. 查看GC日志 2. dump堆内存 3. 使用MAT分析 4. 查看代码中是否存在内存泄漏

### Q8: G1 的 Region 是什么？

**答：** G1将堆划分为多个大小相等的Region（1MB-32MB），每个Region可以独立作为Eden/Survivor/Old。优先回收垃圾最多的Region。

---

# 学习路线图

`
第一阶段：基础（1-2周）
├── 线程创建、状态、方法
├── synchronized 原理
├── volatile 原理
└── CAS 机制

第二阶段：进阶（2-3周）
├── Lock 体系
├── JUC 工具类
├── 线程池
├── 并发容器
└── ThreadLocal

第三阶段：JVM（2-3周）
├── 内存结构
├── 垃圾回收算法
├── 垃圾收集器
├── 类加载机制
└── JVM调优

第四阶段：实战（持续）
├── 刷面试题
├── 看源码（ConcurrentHashMap、线程池）
├── 实际项目应用
└── 性能调优实践
`

---

## 推荐资源

- 《Java并发编程实战》
- 《深入理解Java虚拟机》
- 美团技术博客 JVM系列
- 微信公众号：Java并发编程之美

---
## 推荐资源

- 《Java并发编程实战》
- 《深入理解Java虚拟机》
- 美团技术博客 JVM系列
- 微信公众号：Java并发编程之美

---

> 💡 **学习建议**：
> 1. 先理解原理，再记面试题
> 2. 结合实际场景思考
> 3. 多写代码验证
> 4. 看源码提升深度


---

# 附录A：实战技巧与常见追问

## A.1 synchronized 原理追问

### Q: synchronized 锁住的是对象还是代码？

**答：** synchronized 锁的是对象，不是代码！

`java
public class LockTarget {
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();
    
    public void method1() {
        synchronized (lock1) {  // 锁的是 lock1 对象
            // 只有其他线程也 synchronized(lock1) 才会阻塞
        }
    }
    
    public void method2() {
        synchronized (lock2) {  // 锁的是 lock2 对象
            // 与 method1 不互斥！
        }
    }
}
`

### Q: 锁对象的改变会导致不可重入？

`java
public class LockChange {
    private String lock = "LOCK";  // 锁对象
    
    public synchronized void method1() {
        System.out.println("method1");
        lock = "NEW_LOCK";  // ❌ 改变了锁对象！
        method2();  // 可能获取不到锁
    }
    
    public synchronized void method2() {
        // this 被锁住，所以可以进入
        System.out.println("method2");
    }
}
`

## A.2 线程池最佳实践

### 实践1：使用 ThreadFactory 给线程命名

`java
import java.util.concurrent.*;

public class NamedThreadPool {
    public static void main(String[] args) {
        ThreadFactory factory = new ThreadFactory() {
            private int count = 0;
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                t.setName("my-pool-" + count++);
                return t;
            }
        };
        
        ExecutorService pool = new ThreadPoolExecutor(
            5, 10, 60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(100),
            factory,
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}
`

### 实践2：异常处理

`java
ExecutorService pool = Executors.newFixedThreadPool(3);

// ❌ 错误：execute 提交的任务，异常会导致线程终止
pool.execute(() -> {
    throw new RuntimeException("任务异常");
});

// ✅ 正确：捕获异常或使用 Future
pool.execute(() -> {
    try {
        // 业务逻辑
    } catch (Exception e) {
        // 处理异常
    }
});

// ✅ 使用 Future 获取异常
Future<?> future = pool.submit(() -> {
    throw new RuntimeException("任务异常");
});
try {
    future.get();
} catch (ExecutionException e) {
    System.out.println("任务异常: " + e.getCause());
}
`

### 实践3：合理设置队列大小

`java
public class QueueSizing {
    public static void main(String[] args) {
        // 估算公式：队列大小 = 期望线程数 × 任务平均耗时 / 单任务处理时间
        
        // CPU密集型：队列可以小一些
        int cpuTasks = Runtime.getRuntime().availableProcessors() + 1;
        
        // IO密集型：队列可以大一些
        int ioTasks = Runtime.getRuntime().availableProcessors() * 100;  // 放大倍数
    }
}
`

## A.3 volatile 进阶问题

### Q: volatile 是如何保证可见性的？

**答：** 通过内存屏障（Memory Barrier）

`
volatile 写操作后：
┌─────────────────────────────────────────┐
│ Store Barrier (存储屏障)                │
│ - 强制将 CPU 缓存中的数据写回主内存      │
│ - 禁止屏障前的写操作重排到屏障后          │
└─────────────────────────────────────────┘

volatile 读操作前：
┌─────────────────────────────────────────┐
│ Load Barrier (加载屏障)                  │
│ - 强制从主内存读取数据到 CPU 缓存         │
│ - 禁止屏障后的读操作重排到屏障前          │
└─────────────────────────────────────────┘
`

### Q: volatile 和 synchronized 的区别？

| 对比项 | volatile | synchronized |
|--------|----------|--------------|
| 作用 | 变量 | 方法/代码块 |
| 原子性 | 不保证 | 保证 |
| 可见性 | 保证 | 保证 |
| 有序性 | 保证（禁止重排序） | 保证（单线程内） |
| 性能 | 轻量级 | 重量级 |

## A.4 死锁检测与分析

### 使用 JStack 分析死锁

`ash
# 1. 找到 Java 进程 ID
jps -l
# 输出：12345 com.example.MyApplication

# 2. 导出线程栈
jstack 12345 > thread_dump.txt

# 3. 查看死锁信息
# 搜索 "Found one Java-level deadlock"
`

### JStack 输出示例

`
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting for lock on 0x000000076b400c10 (a java.lang.Object)
  locked by 0x000000076b400c00 (a java.lang.Object)
"Thread-2":
  waiting for lock on 0x000000076b400c00 (a java.lang.Object)
  locked by 0x000000076b400c10 (a java.lang.Object)

Java stack information for the threads listed above:
===================================================
"Thread-1":
    at DeadLockDemo.lambda(DeadLockDemo.java:15)
    - waiting to lock java.lang.Object@76b400c10
    - locked java.lang.Object@76b400c00
`

## A.5 JVM 问题排查案例

### 案例1：内存泄漏排查

`ash
# 1. 添加 JVM 参数
java -Xms512m -Xmx512m -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/tmp/heap.hprof \
     -XX:+PrintGCDetails -Xloggc:/tmp/gc.log \
     -jar app.jar

# 2. 等待 OOM 发生，生成 dump 文件

# 3. 使用 MAT 分析
# 下载：https:// eclipse.org/mat/
# 打开 dump 文件，查看：
# - Dominator Tree：大对象占用
# - Histogram：按类统计对象数量
# - Top Consumers：最大的对象
`

### 案例2：CPU 高排查

`ash
# 1. 找到占用 CPU 高的线程
top -Hp <pid>

# 2. 导出线程栈
jstack <pid> > thread_dump.txt

# 3. 将线程 ID 转为 16 进制
printf "%x\n" <thread_id>

# 4. 在 thread_dump.txt 中搜索该线程
`

### 案例3：GC 频繁排查

`ash
# 1. 查看 GC 日志
cat gc.log | grep "Full GC"

# 2. 分析 GC 原因
jstat -gcutil <pid> 1000

# 3. 常见问题：
# - Minor GC 频繁：Eden 区太小
# - Full GC 频繁：老年代空间不足 / 内存泄漏
# - Metaspace 满：类加载过多
`

---

# 附录B：JVM 调优参数速查表

## B.1 堆内存参数

| 参数 | 说明 | 示例 |
|------|------|------|
| -Xms | 初始堆大小 | -Xms512m |
| -Xmx | 最大堆大小 | -Xmx2g |
| -Xmn | 新生代大小 | -Xmn256m |
| -XX:NewRatio | 新生代/老年代比例 | -XX:NewRatio=2 |
| -XX:SurvivorRatio | Eden/Survivor比例 | -XX:SurvivorRatio=8 |
| -XX:MaxTenuringThreshold | 对象晋升老年代年龄 | -XX:MaxTenuringThreshold=15 |

## B.2 元空间参数

| 参数 | 说明 | 示例 |
|------|------|------|
| -XX:MetaspaceSize | 初始元空间大小 | -XX:MetaspaceSize=128m |
| -XX:MaxMetaspaceSize | 最大元空间大小 | -XX:MaxMetaspaceSize=256m |

## B.3 GC 参数

| 参数 | 说明 | 示例 |
|------|------|------|
| -XX:+UseSerialGC | Serial GC | 客户端模式默认 |
| -XX:+UseParallelGC | Parallel GC | 服务端默认 |
| -XX:+UseConcMarkSweepGC | CMS GC | 老年代并发 |
| -XX:+UseG1GC | G1 GC | 整堆收集 |
| -XX:MaxGCPauseMillis | 最大GC停顿时间 | -XX:MaxGCPauseMillis=200 |
| -XX:G1HeapRegionSize | Region大小 | -XX:G1HeapRegionSize=1m |
| -XX:+HeapDumpOnOutOfMemoryError | OOM时dump | 生产必备 |
| -XX:HeapDumpPath | dump路径 | -XX:HeapDumpPath=/tmp |

## B.4 其他常用参数

| 参数 | 说明 | 示例 |
|------|------|------|
| -Xss | 线程栈大小 | -Xss1m |
| -XX:+PrintGCDetails | 详细GC日志 | 调试用 |
| -XX:+PrintGCDateStamps | GC时间戳 | 方便分析 |
| -Xloggc:file | GC日志文件 | -Xloggc:/tmp/gc.log |
| -XX:+UseStringDeduplication | 字符串去重 | JDK8u20+ |
| -XX:+UseTLAB | 线程本地分配缓冲 | 默认开启 |

---

# 附录C：常见场景模板

## C.1 秒杀系统并发设计

`java
import java.util.concurrent.*;

public class SeckillSystem {
    private final ConcurrentHashMap<String, Integer> stock = new ConcurrentHashMap<>();
    private final Semaphore semaphore = new Semaphore(100);  // 限流100并发
    private final LongAdder successCount = new LongAdder();
    
    public boolean seckill(String userId, String productId) {
        try {
            // 1. 限流
            if (!semaphore.tryAcquire()) {
                return false;  // 请求过多
            }
            
            try {
                // 2. 检查库存（原子操作）
                Integer remain = stock.computeIfAbsent(productId, k -> 100);
                if (remain <= 0) {
                    return false;  // 库存不足
                }
                
                // 3. 扣减库存（CAS）
                while (true) {
                    remain = stock.get(productId);
                    if (remain <= 0) return false;
                    if (stock.replace(productId, remain, remain - 1)) {
                        successCount.increment();
                        return true;  // 秒杀成功
                    }
                    // CAS 失败，重试
                }
            } finally {
                semaphore.release();
            }
        } catch (Exception e) {
            return false;
        }
    }
}
`

## C.2 批量任务处理

`java
import java.util.*;
import java.util.concurrent.*;

public class BatchProcessor {
    private final ExecutorService executor;
    private final int batchSize;
    private final int threadCount;
    
    public BatchProcessor(int batchSize, int threadCount) {
        this.batchSize = batchSize;
        this.threadCount = threadCount;
        this.executor = new ThreadPoolExecutor(
            threadCount, threadCount * 2, 60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(1000),
            r -> {
                Thread t = new Thread(r);
                t.setName("batch-worker-" + t.getId());
                return t;
            },
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
    
    public <T, R> List<R> process(List<T> items, 
                                   Function<T, R> processor) 
            throws InterruptedException, ExecutionException {
        
        List<Future<R>> futures = new ArrayList<>();
        
        for (T item : items) {
            futures.add(executor.submit(() -> processor.apply(item)));
        }
        
        List<R> results = new ArrayList<>();
        for (Future<R> f : futures) {
            results.add(f.get());  // 等待完成
        }
        
        return results;
    }
    
    public void shutdown() {
        executor.shutdown();
    }
}
`

## C.3 延迟任务调度

`java
import java.util.concurrent.*;

public class DelayTaskScheduler {
    private final DelayQueue<DelayedTask> delayQueue = new DelayQueue<>();
    private final ExecutorService worker = Executors.newFixedThreadPool(3);
    
    public DelayTaskScheduler() {
        // 启动消费线程
        for (int i = 0; i < 3; i++) {
            worker.submit(this::consume);
        }
    }
    
    public void schedule(Runnable task, long delayMillis) {
        delayQueue.put(new DelayedTask(task, delayMillis));
    }
    
    private void consume() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                DelayedTask task = delayQueue.take();
                task.run();  // 执行任务
            } catch (InterruptedException e) {
                break;
            }
        }
    }
    
    static class DelayedTask implements Delayed {
        private final Runnable task;
        private final long startTime;
        
        public DelayedTask(Runnable task, long delayMillis) {
            this.task = task;
            this.startTime = System.currentTimeMillis() + delayMillis;
        }
        
        @Override
        public long getDelay(TimeUnit unit) {
            return unit.convert(startTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
        }
        
        @Override
        public int compareTo(Delayed o) {
            return Long.compare(startTime, ((DelayedTask) o).startTime);
        }
        
        public void run() {
            task.run();
        }
    }
}
`

---

> 📝 **文档维护**
> 最后更新：2026-03-25
> 版本：v1.0
> 状态：✅ 完整版

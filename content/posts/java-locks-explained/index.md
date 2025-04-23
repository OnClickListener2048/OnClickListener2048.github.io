---
title: "Java 并发编程之锁：从 Synchronized 到 ReentrantLock，再探乐观与公平"
date: 2025-01-25T14:00:00+08:00 # 修改为实际日期
lastmod: 2025-01-25T14:00:00+08:00 # 修改为实际日期
author: "OnClickListener" # 修改为你的名字
# authorLink: "Your Link" # 可选：你的个人链接
description: "深入理解 Java 中锁的核心概念，详细对比 Synchronized 与 ReentrantLock，并解析悲观锁、乐观锁、公平锁与非公平锁的原理与区别。"
# license: "" # 可选：指定许可证
resources: # 可选：添加特色图片或其他资源
- name: "featured-image"
  src: "java-locks-diagram.png"
tags: ["Java", "并发编程", "锁", "Synchronized", "ReentrantLock", "乐观锁", "悲观锁", "公平锁", "JUC", "多线程"]
categories: ["Java核心", "并发编程"]
# series: ["深入Java并发"] # 可选：添加到系列
# weight: # 可选：用于排序
draft: false # 设置为 false 以便发布
---

## 前言：为何需要锁？

在多线程并发编程中，当多个线程需要访问和修改共享资源（如共享变量、对象、文件等）时，如果没有适当的同步机制，就可能导致数据竞争（Race Condition）、数据不一致等问题。**锁（Lock）** 就是 Java 中最基本、最重要的线程同步机制之一，它用于控制多个线程对共享资源的访问权限，保证在同一时刻只有一个（或有限个）线程能够访问被保护的资源，从而确保线程安全。

本文将带你深入了解 Java 中常见的锁机制和相关概念。

{{< admonition type=tip title="本文目标" open=true >}}
*   理解锁的基本作用。
*   掌握 `synchronized` 关键字的使用和原理。
*   掌握 `ReentrantLock` 的使用和特性。
*   清晰辨析 `synchronized` 与 `ReentrantLock` 的核心区别。
*   理解悲观锁和乐观锁的设计思想。
*   理解公平锁和非公平锁的区别与权衡。
{{< /admonition >}}

## Java 内置锁：`synchronized`

`synchronized` 是 Java 语言层面的关键字，也是最基础、最常用的内置锁。它依赖于 JVM 实现，背后关联着每个 Java 对象都拥有的**监视器锁（Monitor Lock）**，也称为**内部锁（Intrinsic Lock）**。

**使用方式：**

1.  **修饰实例方法:** 锁对象是当前实例 (`this`)。
    ```java
    public synchronized void instanceMethod() {
        // 同步代码块，访问实例资源
    }
    ```
2.  **修饰静态方法:** 锁对象是当前类的 `Class` 对象。
    ```java
    public static synchronized void staticMethod() {
        // 同步代码块，访问静态资源
    }
    ```
3.  **修饰代码块:** 可以显式指定锁对象，更加灵活。
    ```java
    Object lock = new Object();
    public void blockMethod() {
        synchronized (lock) { // 使用指定对象作为锁
            // 同步代码块
        }
        synchronized (this) { // 使用当前实例作为锁
            // 另一个同步代码块
        }
        synchronized (MyClass.class) { // 使用类对象作为锁
            // 访问静态资源的同步代码块
        }
    }
    ```

**`synchronized` 的特性：**

*   **可重入性 (Reentrant):** 同一个线程可以多次获取同一个锁，不会自己把自己锁死。每次获取锁时，计数器加 1；每次释放锁时，计数器减 1。当计数器为 0 时，锁才被完全释放。
*   **非公平性 (Unfair by default):** JVM 不保证等待时间最长的线程一定能优先获得锁。新来的线程可能“插队”先获取到锁，这通常能提高吞吐量，但可能导致某些线程“饿死”（长时间获取不到锁）。
*   **阻塞性:** 获取不到锁的线程会进入阻塞状态，等待锁被释放。
*   **自动释放:** 当线程执行完 `synchronized` 代码块或方法后（无论是正常结束还是异常退出），JVM 会自动释放锁，不易出现死锁（因忘记释放锁导致）。
*   **不可中断:** 等待获取 `synchronized` 锁的线程是不可被中断的（`Thread.interrupt()` 无效）。

{{< admonition type=info title="Synchronized 的底层优化" >}}
JVM 对 `synchronized` 进行了大量优化，如偏向锁、轻量级锁、自旋锁、锁粗化、锁消除等，以减少锁带来的性能开销。在低竞争或无竞争的情况下，其性能可能与 `ReentrantLock` 相当甚至更好。
{{< /admonition >}}

## JUC 锁：`ReentrantLock`

`ReentrantLock` 是 `java.util.concurrent.locks` (JUC) 包下的一个类，它实现了 `Lock` 接口，提供了比 `synchronized` 更强大、更灵活的锁机制。它是基于 AQS (AbstractQueuedSynchronizer) 框架实现的。

**基本使用：**

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockExample {
    private final Lock lock = new ReentrantLock(); // 默认创建非公平锁
    // private final Lock lock = new ReentrantLock(true); // 创建公平锁

    public void performAction() {
        lock.lock(); // 获取锁
        try {
            // --- 临界区：访问共享资源的代码 ---
            System.out.println(Thread.currentThread().getName() + " acquired the lock.");
            // 模拟业务操作
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // 重新设置中断状态
        } finally {
            lock.unlock(); // !!! 必须在 finally 块中释放锁 !!!
            System.out.println(Thread.currentThread().getName() + " released the lock.");
        }
    }
}
```

{{< admonition type=warning title="务必在 finally 中释放 ReentrantLock" >}}
使用 `ReentrantLock` 时，必须手动在 `finally` 块中调用 `unlock()` 方法来释放锁。否则，如果在临界区代码中发生异常，锁将永远不会被释放，导致其他线程永远无法获取锁。
{{< /admonition >}}

**`ReentrantLock` 的特性：**

*   **可重入性:** 与 `synchronized` 一样，是可重入的。
*   **可中断获取锁 (`lockInterruptibly()`):** 等待获取锁的线程可以响应中断请求（`Thread.interrupt()`），避免死等。
*   **可限时获取锁 (`tryLock(long time, TimeUnit unit)`):** 尝试在指定时间内获取锁，超时则放弃，避免无限等待。
*   **尝试非阻塞获取锁 (`tryLock()`):** 立即尝试获取锁，成功返回 `true`，失败返回 `false`，不会阻塞。
*   **公平性可配置:** 可以在构造函数中指定创建公平锁 (`new ReentrantLock(true)`) 或非公平锁（默认）。
*   **条件变量 (`Condition`):** 可以关联多个 `Condition` 对象，实现更精确的线程通知/唤醒机制（类似于 `Object` 的 `wait()/notify()/notifyAll()`，但更强大，可以分组唤醒）。

## `synchronized` vs `ReentrantLock` 对比

| 特性           | `synchronized`                                | `ReentrantLock` (`java.util.concurrent.locks.Lock`) |
| :------------- | :-------------------------------------------- | :-------------------------------------------------- |
| **实现层面**   | Java 关键字，JVM 内置实现                     | Java 类库 (JUC包)，基于 AQS 实现                   |
| **锁的获取/释放** | 隐式，JVM 自动管理                          | 显式，需要手动 `lock()` 和 `unlock()` (必须在 `finally` 中) |
| **可重入性**   | 是                                            | 是                                                  |
| **公平性**     | 非公平（通常）                                | 可选（默认非公平，可配置为公平）                    |
| **获取锁可中断** | 否                                            | 是 (`lockInterruptibly()`)                           |
| **尝试获取锁** | 否                                            | 是 (`tryLock()`, `tryLock(time, unit)`)             |
| **条件变量**   | 单一条件队列 (`wait/notify/notifyAll`)        | 可关联多个 `Condition` 对象，更灵活的等待/通知     |
| **性能**       | 经 JVM 优化后，在低竞争下性能可能很好         | 早期版本性能优势明显，现在与优化后的 `synchronized` 差距缩小，但在高竞争或需要高级特性时仍有优势 |
| **使用便捷性** | 简单，不易出错（自动释放）                    | 相对复杂，需要手动释放锁，易忘 `unlock()` 导致问题    |

**选择建议：**

*   **优先 `synchronized`:** 在锁竞争不激烈、功能要求简单（不需要中断、尝试、公平性、多条件变量）的情况下，优先使用 `synchronized`，代码更简洁，不易出错。
*   **选择 `ReentrantLock`:** 当你需要以下高级功能时：
    *   可中断的锁获取。
    *   尝试非阻塞或限时获取锁。
    *   需要公平锁机制。
    *   需要使用多个条件变量进行精细的线程通信。
    *   在非常高的竞争下，可能需要其更可控的性能表现（需实测）。

## 悲观锁 vs 乐观锁

这是两种不同的并发控制思想。

*   **悲观锁 (Pessimistic Locking):**
    *   **思想:** 总是假设最坏的情况，认为数据在被访问时**总会**发生冲突（被其他线程修改）。所以在访问数据**之前**就加锁，阻止其他线程访问，访问完成后再解锁。
    *   **实现:** `synchronized` 和 `ReentrantLock` 都属于典型的悲观锁实现。数据库中的行锁、表锁也属于悲观锁。
    *   **优点:** 实现简单，能有效防止数据冲突。
    *   **缺点:** 在冲突实际很少发生的情况下，加锁、解锁以及线程阻塞/唤醒的开销较大，可能导致系统吞吐量下降。

*   **乐观锁 (Optimistic Locking):**
    *   **思想:** 总是假设最好的情况，认为数据在被访问时**通常不会**发生冲突。它在访问数据时不加锁，而是在**更新**数据时去判断，在此期间数据有没有被其他线程修改过。如果没被修改，则成功更新；如果被修改了，则更新失败，然后采取其他策略（如重试、报错等）。
    *   **实现:** 通常通过以下机制实现：
        *   **版本号机制 (Versioning):** 在数据表中增加一个版本号字段。读取数据时，连同版本号一起读出。更新时，比较当前版本号与读取时的版本号是否一致，一致则更新数据并将版本号加 1，不一致则表示数据已被修改，更新失败。
        *   **CAS 操作 (Compare-and-Swap):** 这是 CPU 指令级别的原子操作。`java.util.concurrent.atomic` 包下的原子类（如 `AtomicInteger`, `AtomicReference`）以及 AQS 框架内部都广泛使用了 CAS。CAS 操作包含三个操作数：内存位置 V、预期原值 A、新值 B。当且仅当 V 的值等于 A 时，才将 V 的值原子性地更新为 B，否则不做任何操作。更新失败后通常会进行自旋重试。
    *   **优点:** 在冲突较少（读多写少）的场景下，避免了加锁解锁的开销，提高了吞吐量。
    *   **缺点:** 实现相对复杂。如果冲突频繁发生，重试的代价可能会超过悲观锁的开销。存在 ABA 问题（需要使用 `AtomicStampedReference` 等解决）。

## 公平锁 vs 非公平锁

这是针对锁获取机制的分类，主要体现在等待队列中的线程获取锁的顺序。

*   **公平锁 (Fair Lock):**
    *   **规则:** 遵循先来后到（FIFO）的原则。等待时间最长的线程将优先获得锁。
    *   **实现:** `ReentrantLock(true)` 可以创建公平锁。
    *   **优点:** 可以避免线程“饿死”，保证了所有线程都有机会获得锁。
    *   **缺点:** 为了维护顺序，需要进行额外的管理和线程切换，导致系统吞吐量通常低于非公平锁。

*   **非公平锁 (Unfair Lock):**
    *   **规则:** 不保证先来后到。当锁被释放时，等待队列中的线程和新尝试获取锁的线程会进行竞争，新来的线程有可能“插队”成功。
    *   **实现:** `synchronized` 默认是非公平的。`ReentrantLock()` 默认创建非公平锁。
    *   **优点:** 减少了线程唤醒和切换的开销（因为刚释放锁的线程可能立刻再次获取锁，或者新来的线程可以直接获取），通常具有更高的系统吞吐量。
    *   **缺点:** 可能导致等待队列中的某些线程长时间获取不到锁（线程“饿死”）。

**选择：**

*   除非有明确的公平性需求（例如，业务逻辑要求严格的顺序处理），否则**通常推荐使用非公平锁**，因为它能带来更高的性能。
*   `ReentrantLock` 提供了选择的灵活性。

## 总结

{{< admonition type=milestone title="核心要点回顾" open=true >}}
*   **锁**是解决并发访问共享资源问题的关键同步机制。
*   **`synchronized`** 是 Java 内置锁，简单易用，自动管理，但功能相对基础（非公平、不可中断）。JVM 优化使其在很多场景下性能良好。
*   **`ReentrantLock`** 是 JUC 提供的更灵活的锁，提供可中断、可尝试、公平性选择、多条件变量等高级功能，但需要手动释放锁（`finally` 块）。
*   **悲观锁**（如 `synchronized`, `ReentrantLock`）假设冲突，先加锁后访问，适用于写操作多、冲突概率高的场景。
*   **乐观锁**（如 CAS, 版本号）假设无冲突，更新时检查，适用于读操作多、冲突概率低的场景，可提高吞ب量。
*   **公平锁**保证 FIFO，避免饿死，但牺牲吞吐量；**非公平锁**允许插队，吞吐量更高，但可能导致饿死。默认和推荐通常是非公平锁。
{{< /admonition >}}

理解并恰当使用 Java 中的各种锁机制，是编写健壮、高效并发程序的基石。需要根据具体的业务场景、性能需求以及对公平性的要求来选择最合适的锁策略。
---
title: "Kotlin Flow 快速入门"
subtitle: ""
date: 2025-04-10T23:36:35+08:00
lastmod: 2025-04-10T23:36:35+08:00
draft: false
author: ""
authorLink: ""
description: ""
license: ""
images: []

tags: []
categories: ["Kotlin"]

featuredImage: "flow.webp"
featuredImagePreview: "flow.webp"
resources:
  - name: "flow"
    src: "flow.webp"

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: true
ruby: true
fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: false

toc:
  auto: false
code:
  copy: true
  maxShownLines: 50
math:
  enable: false
  # ...
mapbox:
  # ...
share:
  enable: true
  # ...
comment:
  enable: true
  # ...
library:
  css:
    # someCSS = "some.css"
    # located in "assets/"
    # Or
    # someCSS = "https://cdn.example.com/some.css"
  js:
    # someJS = "some.js"
    # located in "assets/"
    # Or
    # someJS = "https://cdn.example.com/some.js"
seo:
  images: []
  # ...
---

```markdown
# Kotlin Flow 快速入门

Kotlin Flow 是 Kotlin 协程库提供的一种用于处理**异步数据流**的类型。如果你熟悉 RxJava/RxKotlin，你会发现 Flow 的概念与之类似，但它基于 Kotlin 协程构建，提供了更简洁、更符合语言习惯的 API，并能更好地与结构化并发集成。

可以将 Flow 想象成一个异步版本的 `Sequence` 或 `Iterator`。它按需生产（emit）一系列值，而消费者（collector）则异步地处理这些值。

## 为什么需要 Flow？

在现代应用程序开发中，处理异步事件流非常常见：

1.  **用户界面事件**：按钮点击、文本输入变化等。
2.  **网络请求**：获取可能分块或随时间更新的数据。
3.  **数据库访问**：监听数据库变化并获取更新。
4.  **传感器数据**：连续接收设备传感器读数。

Flow 提供了一种统一的方式来处理这些场景，具有以下优点：

*   **基于协程**：天然支持挂起函数，与 Kotlin 的异步模型无缝集成。
*   **结构化并发**：Flow 的生命周期通常与启动它的协程作用域绑定，易于管理和取消。
*   **冷流 (Cold Streams)**：默认情况下，Flow 是冷的。意味着只有当有消费者开始收集（collect）时，生产者代码才会执行。每个新的收集者都会触发一次新的执行。
*   **丰富的操作符**：提供了大量类似于 RxJava 的操作符（如 `map`, `filter`, `zip`, `flatMapConcat` 等）用于转换和组合流。
*   **背压支持**：Flow 通过协程的挂起机制天然支持背压，生产者不会压垮消费者。

## Flow 的核心概念

1.  **Flow<T>**：表示一个异步数据流的接口，它能按顺序发出 (emit) 类型为 `T` 的零个或多个值。
2.  **生产者 (Producer)**：负责**生产**或**发出**数据的代码块。通常使用 `flow { ... }` 构建器创建。
3.  **消费者 (Collector)**：负责**接收**和**处理**数据的代码块。通过调用**末端操作符**（如 `collect`）来触发 Flow 的执行。
4.  **中间操作符 (Intermediate Operators)**：如 `map`, `filter`, `onEach` 等。它们应用于上游 Flow 并返回一个新的下游 Flow。这些操作符本身**不会**触发 Flow 的执行，它们是**惰性**的。
5.  **末端操作符 (Terminal Operators)**：如 `collect`, `toList`, `first`, `single`, `reduce`, `fold` 等。它们是**挂起函数**，会启动 Flow 的收集过程，并等待其完成。

## 创建 Flow

有多种方式可以创建 Flow：

**1. 使用 `flow { ... }` 构建器 (最常用)**

这是最灵活的方式，你可以在 `flow` 代码块中使用 `emit()` 函数来发出值。`emit` 本身不是挂起函数，但 `collect` 是挂起的。`flow` 代码块内的代码直到被收集时才会执行。

```kotlin
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.runBlocking

// 模拟一个每 100ms 发出一个数字的 Flow
fun simpleFlow(): Flow<Int> = flow {
    println("Flow started")
    for (i in 1..3) {
        delay(100) // 模拟耗时操作
        println("Emitting $i")
        emit(i) // 发出值
    }
}

fun main() = runBlocking {
    println("Calling simpleFlow()...")
    val flow = simpleFlow()

    println("Calling collect...")
    flow.collect { value ->
        println("Collected $value")
    }
    println("Collect finished.")

    // 注意：如果再次 collect，Flow 会重新执行
    println("\nCalling collect again...")
    flow.collect { value ->
        println("Collected again $value")
    }
    println("Second collect finished.")
}
```

**输出:**
```
Calling simpleFlow()...
Calling collect...
Flow started
Emitting 1
Collected 1
Emitting 2
Collected 2
Emitting 3
Collected 3
Collect finished.

Calling collect again...
Flow started // Flow 重新开始执行
Emitting 1
Collected again 1
Emitting 2
Collected again 2
Emitting 3
Collected again 3
Second collect finished.
```

**2. 使用 `flowOf(...)`**

用于从固定数量的值创建 Flow。

```kotlin
import kotlinx.coroutines.flow.flowOf
import kotlinx.coroutines.flow.collect
import kotlinx.coroutines.runBlocking

fun main() = runBlocking {
    flowOf(1, 2, 3, "four", "five")
        .collect { value ->
            println("Collected: $value")
        }
}
```

**3. 使用 `.asFlow()` 扩展函数**

可以将集合、序列、范围等转换为 Flow。

```kotlin
import kotlinx.coroutines.flow.asFlow
import kotlinx.coroutines.flow.collect
import kotlinx.coroutines.runBlocking

fun main() = runBlocking {
    // 从 List 创建
    listOf(1, 2, 3).asFlow()
        .collect { println("From List: $it") }

    println("---")

    // 从 IntRange 创建
    (1..5).asFlow()
        .collect { println("From Range: $it") }
}
```

## 消费 Flow (使用末端操作符)

Flow 需要通过**末端操作符**来消费。最常见的末端操作符是 `collect`。

*   **`collect { value -> ... }`**：这是一个挂起函数，它会启动 Flow 的执行，并对每个发出的值执行给定的 lambda 表达式。

```kotlin
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.runBlocking

fun main() = runBlocking {
    (1..5).asFlow()
        .filter { it % 2 == 0 } // 中间操作符：只保留偶数
        .map { "Value: $it" } // 中间操作符：转换为字符串
        .collect { // 末端操作符：触发执行并打印
            println(it)
        }
}
```

**输出:**
```
Value: 2
Value: 4
```

## Flow 是冷的 (Cold Streams)

如前所述，Flow 是冷的。这意味着：

*   Flow 构建块 (`flow { ... }`) 或中间操作符（如 `map`, `filter`）本身不执行任何操作。
*   只有当调用末端操作符（如 `collect`）时，Flow 才开始执行其生产者逻辑。
*   每次调用末端操作符都会**重新**执行整个 Flow 链（除非使用了 `shareIn` 或 `stateIn` 转换为热流，但这超出了快速入门的范围）。

## 常用操作符示例

Flow 提供了丰富的操作符来处理数据流。

*   **`map`**: 转换每个元素。
*   **`filter`**: 过滤满足条件的元素。
*   **`onEach`**: 对每个元素执行一个副作用（如打印日志），不改变元素本身。
*   **`take`**: 只取前 N 个元素。
*   **`zip`**: 将两个 Flow 的元素按顺序配对。
*   **`flatMapConcat` / `flatMapMerge` / `flatMapLatest`**: 将每个元素映射到一个新的 Flow，并将这些 Flow 合并。

```kotlin
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.runBlocking

fun main() = runBlocking {
    (1..5).asFlow()
        .onEach { delay(50) } // 模拟处理延迟
        .filter {
            println("Filtering $it")
            it > 2
        }
        .map {
            println("Mapping $it")
            "Mapped: $it"
        }
        .take(2) // 只取转换后的前两个结果
        .collect {
            println("Collected $it")
        }
}
```

**输出 (大致顺序，delay 会影响精确时序):**
```
Filtering 1
Filtering 2
Filtering 3
Mapping 3
Collected Mapped: 3
Filtering 4
Mapping 4
Collected Mapped: 4
```
(注意：因为 `take(2)`，一旦收集到两个元素，Flow 就会停止，所以 5 不会被处理。)

## 异常处理

使用 `catch` 操作符来捕获上游 Flow 中发生的异常。`catch` 只能捕获其**上游**的异常。

```kotlin
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.runBlocking

fun failingFlow(): Flow<Int> = flow {
    emit(1)
    emit(2)
    throw RuntimeException("Something went wrong!")
    // emit(3) // 这行不会执行
}

fun main() = runBlocking {
    failingFlow()
        .map { "Value: $it" }
        .catch { e ->
            println("Caught exception: ${e.message}")
            // 可以选择发出一个默认值
            emit("Error occurred")
        }
        .collect { println(it) }
}
```

**输出:**
```
Value: 1
Value: 2
Caught exception: Something went wrong!
Error occurred
```

## 完成处理

使用 `onCompletion` 操作符来指定当 Flow 完成（无论是正常完成还是因异常完成）时执行的操作。它通常用于资源清理。

```kotlin
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.runBlocking

fun main() = runBlocking {
    (1..3).asFlow()
        .onCompletion { cause ->
            if (cause == null) {
                println("Flow completed successfully")
            } else {
                println("Flow completed with exception: ${cause.message}")
            }
        }
        .collect { println("Collected $it") }

    println("---")

    // 示例：带异常的 Flow
    failingFlow()
        .catch { /* 捕获异常以防止崩溃，但 onCompletion 仍然会收到 cause */ }
        .onCompletion { cause ->
             if (cause == null) {
                println("Flow completed successfully")
            } else {
                // 注意：这里的 cause 是 catch 操作符处理之前的原始异常
                println("Flow completed with exception: ${cause.message}")
            }
        }
        .collect { println("Collected $it") }
}
```

**输出:**
```
Collected 1
Collected 2
Collected 3
Flow completed successfully
---
Collected 1
Collected 2
Flow completed with exception: Something went wrong!
```

## 切换上下文 (Context Switching)

默认情况下，Flow 的生产者代码运行在收集者所在的协程上下文中。可以使用 `flowOn()` 操作符来改变**上游**操作（包括 `flow` 构建块和之前的中间操作符）执行的 `CoroutineDispatcher`。

这对于将 CPU 密集型或 IO 密集型操作切换到合适的线程池非常有用。

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun simpleContextFlow(): Flow<Int> = flow {
    log("Flow started")
    for (i in 1..3) {
        delay(100)
        log("Emitting $i")
        emit(i)
    }
}.flowOn(Dispatchers.IO) // <<< 改变 flow 构建块的执行上下文

fun main() = runBlocking {
    log("Starting collection")
    simpleContextFlow()
        .map { // 这个 map 仍然在下游 (收集者) 的上下文中执行
            log("Mapping $it")
            it * 2
        }
        // .flowOn(Dispatchers.Default) // 如果需要，可以再次切换 map 的上下文
        .collect { // collect 运行在 runBlocking 的上下文中 (通常是 main 线程)
            log("Collected $it")
        }
    log("Collection finished")
}
```

**输出 (线程名可能不同):**
```
[main @coroutine#1] Starting collection
[DefaultDispatcher-worker-1 @coroutine#2] Flow started  // <<< 在 IO 线程池
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 1
[main @coroutine#1] Mapping 1                      // <<< map 在 main 线程
[main @coroutine#1] Collected 2                    // <<< collect 在 main 线程
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 2
[main @coroutine#1] Mapping 2
[main @coroutine#1] Collected 4
[DefaultDispatcher-worker-1 @coroutine#2] Emitting 3
[main @coroutine#1] Mapping 3
[main @coroutine#1] Collected 6
[main @coroutine#1] Collection finished
```

## 总结

Kotlin Flow 是处理异步数据流的强大工具。本快速入门介绍了：

*   Flow 的基本概念：异步、冷流。
*   如何创建 Flow (`flow`, `flowOf`, `asFlow`)。
*   如何使用末端操作符消费 Flow (`collect`)。
*   常用的中间操作符 (`map`, `filter`)。
*   异常处理 (`catch`) 和完成处理 (`onCompletion`)。
*   使用 `flowOn` 进行上下文切换。

这只是 Flow 功能的冰山一角。深入学习可以探索更多高级操作符、缓冲策略、热流 (`SharedFlow`, `StateFlow`) 以及它们在 Android 开发等场景中的应用。

```

<!--more-->

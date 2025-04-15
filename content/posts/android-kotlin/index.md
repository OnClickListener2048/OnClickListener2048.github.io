---
weight: 1
title: "Kotlin Flow 快速入门"
date: 2019-10-01T17:55:28+08:00
lastmod: 2019-10-01T17:55:28+08:00
draft: false
author: "Watson"
authorLink: "https://dillonzq.com"
description: "Kotlin Flow 是 Kotlin 协程库提供的一种用于处理**异步数据流**的类型。"
images: []
resources:
  - name: "featured-image"
    src: "featured-image.webp"

tags: ["Kotlin", "Flow", "Android", "协程", "异步编程", "入门", "Jetpack"] # 文章标签
categories: ["Android开发", "Kotlin编程"] # 文章分类

twemoji: false
lightgallery: true
---

## 1. 引言：拥抱异步数据流的新方式

在现代 Android 开发中，处理网络请求、数据库访问、用户交互等**异步**操作是家常便饭。如何优雅、高效地管理这些随时间产生的数据流，一直是开发者关注的焦点。Kotlin 协程 (Coroutines) 为我们带来了强大的异步编程模型，而 **Kotlin Flow** 正是构建于协程之上的、用于处理**冷数据流 (Cold Streams)** 的解决方案。

如果你曾受困于回调地狱 (Callback Hell)，觉得 LiveData 在某些场景下不够灵活，或者正在为你的 Kotlin 项目寻找 RxJava 的替代品，那么 Flow 将是你理想的选择。

本文将带你：

*   理解 Flow 的核心概念。
*   学习如何创建、转换和收集 Flow。
*   掌握在 Android ViewModel 和 UI 中安全使用 Flow 的最佳实践。

> **学习前提:** 本文假设你已具备 Kotlin 基础语法和 Kotlin 协程的基本知识。

## 2. 为什么选择 Kotlin Flow？

*   **基于协程:** 与 Kotlin 协程深度集成，享受结构化并发带来的便利，简化异步代码管理和生命周期控制。
*   **冷流特性:** Flow 默认是“冷”的，代码块只在被收集 (`collect`) 时执行，有效节省资源。
*   **操作符丰富:** 提供大量类似 RxJava 的操作符 (`map`, `filter`, `flatMapConcat`, `zip` 等)，方便地转换和组合数据。
*   **背压支持:** 内建支持背压 (Backpressure)，能自动处理数据生产和消费速率不匹配的问题。
*   **简洁的错误处理:** 可使用标准 `try-catch` 或 Flow 提供的 `catch` 操作符优雅处理异常。
*   **Jetpack 友好:** 与 ViewModel、Lifecycle 等 Jetpack 组件无缝集成。

## 3. Flow 核心概念解析

可以把 Flow 想象成一个异步的数据序列，就像河流一样，数据项按顺序流动。

{{< admonition type=info title="核心组成" open=true >}}
*   **生产者 (Producer):** 负责产生数据，通常在 `flow { ... }` 构建器内部使用 `emit()` 发射数据。
*   **中间操作符 (Intermediate Operators):** 对数据流进行转换、过滤等操作（如 `map`, `filter`），返回一个新的 Flow。它们是惰性的。
*   **收集者 (Collector) / 终端操作符 (Terminal Operator):** 触发 Flow 的执行并消费数据，最常用的是 `collect`。它是挂起函数。
{{< /admonition >}}

### 3.1 创建 Flow (生产者)

最基础的创建方式是使用 `flow { ... }` 构建器：

```kotlin
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.delay

// 定义一个 Flow，每秒发射一个数字 (0, 1, 2)
fun simpleNumberFlow(): Flow<Int> = flow {
    println("Flow started") // 只有 collect 时才会打印
    for (i in 0..2) {
        delay(1000) // 模拟耗时操作
        println("Emitting $i")
        emit(i) // 发射数据项
    }
}
Use code with caution.
Markdown
其他便捷构建器：
flowOf(item1, item2, ...): 从固定值创建。
listOf(1, 2, 3).asFlow(): 从集合或序列转换。
3.2 消费 Flow (终端操作符)
使用终端操作符来启动 Flow 并接收数据。collect 是最常用的：
import kotlinx.coroutines.runBlocking
import kotlinx.coroutines.flow.collect

fun main() = runBlocking { // 启动一个协程环境来运行 suspend 函数
    println("Calling flow...")
    val numberFlow = simpleNumberFlow()

    println("Calling collect...")
    // collect 是挂起函数，会等待 Flow 完成
    numberFlow.collect { value ->
        println("Collected $value")
    }
    println("Flow collection finished.")
}
Use code with caution.
Kotlin
输出:
Calling flow...
Calling collect...
Flow started
Emitting 0
Collected 0
Emitting 1
Collected 1
Emitting 2
Collected 2
Flow collection finished.
Use code with caution.
Text
其他终端操作符如 toList(), first(), reduce() 等，它们会收集整个 Flow 并返回一个单一值。
3.3 转换 Flow (中间操作符)
中间操作符可以链式调用，对数据流进行加工：
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.flow.filter

suspend fun processData(data: Int): String {
    delay(500) // 模拟处理耗时
    return "Processed data: $data"
}

fun main() = runBlocking {
    simpleNumberFlow() // 原始流: 0, 1, 2
        .filter { it % 2 == 0 } // 过滤奇数: 0, 2
        .map { data -> processData(data) } // 转换数据: "Processed data: 0", "Processed data: 2"
        .collect { result ->
            println(result)
        }
}
Use code with caution.
Kotlin
输出:
Flow started
Emitting 0
Processed data: 0
Emitting 1
Emitting 2
Processed data: 2
Use code with caution.
Text
常用中间操作符：map, filter, transform, take, onEach, debounce, flatMapConcat, zip, combine 等。
4. 在 Android 中实战 Flow
通常在 Repository -> ViewModel -> UI 架构中使用 Flow。
4.1 ViewModel 层：业务逻辑与状态管理
ViewModel 负责调用 Repository 获取数据（通常返回 Flow），处理业务逻辑，并将最终状态暴露给 UI。推荐使用 StateFlow 或 SharedFlow 向 UI 暴露状态。
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import android.util.Log

// 假设的 Repository 和数据类
data class UiState(val isLoading: Boolean = false, val data: String? = null, val error: String? = null)
class DataRepository {
    fun fetchData(): Flow<String> = flow {
        delay(2000) // 模拟网络请求
        // emit("Data fetched successfully!")
        throw RuntimeException("Network error!") // 模拟错误
    }
}

class MyViewModel(private val repository: DataRepository) : ViewModel() {

    // 使用 MutableStateFlow 管理 UI 状态
    private val _uiState = MutableStateFlow(UiState(isLoading = true)) // 初始状态为加载中
    val uiState: StateFlow<UiState> = _uiState.asStateFlow() // 暴露只读的 StateFlow 给 UI

    init {
        loadData()
    }

    fun loadData() {
        _uiState.value = UiState(isLoading = true) // 开始加载，更新状态

        viewModelScope.launch { // 在 ViewModel 的协程作用域中启动
            repository.fetchData()
                .map { data -> UiState(isLoading = false, data = data) } // 成功，更新状态
                .catch { e -> emit(UiState(isLoading = false, error = e.message ?: "Unknown error")) } // 捕获异常，更新状态
                .collect { state ->
                    _uiState.value = state // 将最终状态发射给 StateFlow
                    Log.d("MyViewModel", "State updated: $state")
                }
        }
    }
}
Use code with caution.
Kotlin
{{< admonition type=tip title="StateFlow vs SharedFlow" >}}
StateFlow: 持有最新状态值，新收集者会立即收到最新值。非常适合表示 UI 状态。只有一个值。
SharedFlow: 可以配置重播缓存 (replay)，允许多个收集者接收数据流事件（一对多）。可以发射多个值。
{{< /admonition >}}
4.2 UI 层 (Activity/Fragment)：安全地收集 Flow
关键在于生命周期感知。避免在 UI 不可见时收集 Flow，防止资源浪费和内存泄漏。推荐使用 repeatOnLifecycle API。
import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import android.widget.TextView
import androidx.activity.viewModels // KTX 库，方便获取 ViewModel
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import kotlinx.coroutines.launch
import com.example.yourapp.R // 假设你的 R 文件路径

class MainActivity : AppCompatActivity() {

    private val viewModel: MyViewModel by viewModels() // 获取 ViewModel 实例
    private lateinit var textView: TextView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main) // 假设布局中有个 TextView
        textView = findViewById(R.id.my_text_view)

        // 使用 lifecycleScope + repeatOnLifecycle 安全地收集 Flow
        lifecycleScope.launch {
            // 当生命周期至少为 STARTED 时，执行 collect 代码块
            // 当生命周期进入 STOPPED 时，自动取消 collect
            // 当生命周期再次回到 STARTED 时，重新启动 collect
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    // 在这里根据 State 更新 UI
                    if (state.isLoading) {
                        textView.text = "Loading..."
                    } else if (state.error != null) {
                        textView.text = "Error: ${state.error}"
                    } else {
                        textView.text = state.data ?: "No data"
                    }
                }
            }
        }

        // (可选) 添加一个按钮来触发重新加载
        // val button = findViewById<Button>(R.id.reload_button)
        // button.setOnClickListener { viewModel.loadData() }
    }
}
Use code with caution.
Kotlin
{{< admonition type=warning title="避免直接在 lifecycleScope.launch 中 collect" >}}
直接使用 lifecycleScope.launch { viewModel.flow.collect { ... } } 会导致即使 UI 进入后台 (STOPPED)，Flow 仍然在收集，浪费资源。repeatOnLifecycle 解决了这个问题。
{{< /admonition >}}
4.3 错误处理 (catch 操作符)
catch 操作符能捕获其上游（之前的操作符和 flow 构建器）的异常。它本身也是一个中间操作符。
fun main() = runBlocking {
    flow {
        emit(1)
        emit(2)
        throw RuntimeException("Error on emission!") // 上游异常
        emit(3) // 这不会被发射
    }
    .map { it * 2 } // 上游操作
    .catch { e -> // 捕获上游异常
        println("Caught error: ${e.message}")
        emit(-1) // 可以发射一个默认值或执行其他操作
    }
    .collect { value -> // 下游消费
        println("Collected value: $value")
        // 如果 collect 内部抛出异常，catch 是捕获不到的
    }
}
Use code with caution.
Kotlin
输出:
Collected value: 2
Collected value: 4
Caught error: Error on emission!
Collected value: -1
Use code with caution.
Text
5. Flow vs LiveData vs RxJava
特性	Kotlin Flow	LiveData	RxJava (2/3)
基础	Kotlin 协程	Android Jetpack	独立库 (Java)
类型	冷流 (默认), 热流 (Shared/State)	热流 (生命周期感知)	冷流 (Observable), 热流 (Subject)
背压	内建支持	不支持 (主要用于 UI 状态)	支持 (多种策略)
操作符	丰富	有限 (主要靠 Transformations)	非常丰富
错误处理	try-catch, catch 操作符	通常在 Observer 中处理	onError 回调, 操作符
生命周期感知	需要手动处理 (e.g., repeatOnLifecycle)	内建	需要手动处理 (e.g., dispose)
平台	Kotlin Multiplatform	Android	Java (Android, Server 等)
学习曲线	中等 (需懂协程)	低	高
选择建议:
新 Android 项目/纯 Kotlin 项目: 优先考虑 Kotlin Flow，尤其是 StateFlow 用于 UI 状态。
简单 UI 状态更新: LiveData 仍然是一个简单有效的选择。
已有大量 RxJava 代码的项目: 迁移成本较高，可考虑继续使用 RxJava 或逐步迁移。
6. 总结与展望
Kotlin Flow 为 Android 开发带来了更现代、更简洁、更安全的异步数据流处理方案。通过理解其核心概念（冷流、构建器、操作符、收集器）、掌握与 ViewModel 和 UI 生命周期的结合方式 (viewModelScope, StateFlow, repeatOnLifecycle)，以及熟悉其错误处理机制，你将能更高效地构建响应式、健壮的 Android 应用。
这只是 Flow 的入门，它还有更多高级特性如 SharedFlow、缓冲 (buffer)、并发 (flatMapMerge) 等待你去探索。开始在你的项目中使用 Flow 吧！
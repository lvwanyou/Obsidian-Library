在 Kotlin 协程中，`coroutineScope` 和 `supervisorScope` 都是用于**结构化并发**的作用域构建器，但它们在**错误传播**和**子协程生命周期管理**上有本质区别。以下是它们的核心差异及使用场景：

---

### 1. **核心区别总结**

|**特性**|`coroutineScope`|`supervisorScope`|
|---|---|---|
|**错误传播机制**|子协程失败会**取消父作用域和其他子协程**|子协程失败**不影响父作用域和兄弟协程**|
|**适用场景**|需要原子性操作（全部成功或全部取消）|需要独立任务（部分失败不影响其他任务）|
|**底层 Job 类型**|使用普通 `Job`（严格父子关系）|使用 `SupervisorJob`（宽松父子关系）|
|**典型用例**|并发请求必须全部成功|日志上报、独立后台任务|

---

### 2. **行为对比详解**

#### **(1) 错误处理示例**

kotlin

// 示例 1: coroutineScope - 原子性失败
suspend fun atomicTask() = coroutineScope {
    launch { 
        delay(100)
        throw RuntimeException("Child 1 Failed!") // 会取消另一个子协程
    }
    launch { 
        delay(200)
        println("Child 2") // 不会执行
    }
}
// 结果: 两个子协程都会被取消

// 示例 2: supervisorScope - 独立失败
suspend fun independentTask() = supervisorScope {
    launch { 
        delay(100)
        throw RuntimeException("Child 1 Failed!") // 不影响其他子协程
    }
    launch { 
        delay(200)
        println("Child 2") // 正常执行
    }
}
// 结果: 只有第一个子协程失败，第二个继续完成

#### **(2) 生命周期控制**

- **`coroutineScope`**：  
    任一子协程失败会立即取消整个作用域，适合需要“全有或全无”的场景（如转账操作）。
    
- **`supervisorScope`**：  
    子协程失败不会波及其他任务，适合不相关的后台任务（如同时上传多个日志文件）。
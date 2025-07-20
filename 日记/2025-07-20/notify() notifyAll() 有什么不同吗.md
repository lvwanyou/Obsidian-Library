### **`notify()` 与 `notifyAll()` 的核心区别**

在 Java 的线程协作中，`notify()` 和 `notifyAll()` 都是用于唤醒因调用 `wait()` 而阻塞的线程，但它们的**唤醒范围**和**使用场景**有本质差异。以下是详细对比：

---

## **1. 核心区别总结**

|**特性**|**`notify()`**|**`notifyAll()`**|
|---|---|---|
|**唤醒范围**|随机唤醒 **一个** 等待线程|唤醒 **所有** 等待线程|
|**适用场景**|单消费者或明确知道只需唤醒一个线程|多消费者或条件复杂需全部重新检查|
|**性能影响**|更高（减少不必要的唤醒）|更低（可能引发无效竞争）|
|**安全性**|易导致线程饥饿（需谨慎使用）|更公平，避免线程永远等待|

---

## **2. 底层机制**

### **(1) 等待队列模型**

- 当线程调用 `wait()` 时，会进入对象的 **等待队列（Wait Set）**。
    
- `notify()` 从队列中随机选择一个线程唤醒，`notifyAll()` 唤醒所有线程。
    

java

synchronized (lock) {
    while (conditionNotMet) {
        lock.wait(); // 线程进入等待队列
    }
    // 被唤醒后继续执行
}

### **(2) 唤醒后的行为**

- 被唤醒的线程会 **重新竞争锁**，只有获取锁的线程能继续执行。
    
- 由于 `wait()` 通常在循环中调用（防止虚假唤醒），线程会再次检查条件。
    

---

## **3. 典型场景对比**

### **(1) 生产者-消费者模型（单消费者）**

- **`notify()` 更高效**：  
    生产者只需唤醒一个消费者线程，避免无效竞争。
    

java

// 生产者代码
synchronized (queue) {
    queue.add(item);
    queue.notify(); // 唤醒一个消费者
}

### **(2) 多条件等待（复杂协作）**

- **必须用 `notifyAll()`**：  
    多个线程等待不同条件（如线程A等数据，线程B等空间），需全部唤醒重新检查。
    

java

synchronized (lock) {
    capacity = MAX;
    lock.notifyAll(); // 所有等待线程重新检查条件
}

---

## **4. 常见误区**

### **(1) 误用 `notify()` 导致线程饥饿**

java

// 错误示例：可能让某些线程永远无法被唤醒
synchronized (lock) {
    if (condition) {
        lock.notify(); // 只唤醒一个，其他线程可能永远等待
    }
}

### **(2) 忽略条件检查**

即使使用 `notifyAll()`，也必须用 `while` 循环检查条件：

java

synchronized (lock) {
    while (!condition) { // 必须用 while，不能用 if！
        lock.wait();
    }
}

---

## **5. 性能与公平性权衡**

|**方法**|**优点**|**缺点**|
|---|---|---|
|`notify()`|减少竞争，提高吞吐量|可能引发线程饥饿|
|`notifyAll()`|保证公平性，避免线程永远等待|大量线程竞争锁时性能下降|

**最佳实践**：

- 如果所有等待线程是**同质**的（如多个消费者线程行为相同），优先用 `notify()`。
    
- 如果线程等待**不同条件**，或无法确定唤醒哪个线程，必须用 `notifyAll()`。
    

---

## **6. 代码示例**

### **(1) 正确使用 `notify()`（单消费者）**

java

// 生产者
synchronized (queue) {
    queue.add(item);
    queue.notify(); // 仅唤醒一个消费者
}

// 消费者
synchronized (queue) {
    while (queue.isEmpty()) {
        queue.wait();
    }
    queue.remove();
}

### **(2) 必须用 `notifyAll()`（多条件等待）**

java

// 线程A：等待数据
synchronized (lock) {
    while (!hasData) {
        lock.wait();
    }
}

// 线程B：等待空间
synchronized (lock) {
    while (!hasSpace) {
        lock.wait();
    }
}

// 触发条件变化时
synchronized (lock) {
    hasData = true;
    hasSpace = true;
    lock.notifyAll(); // 必须唤醒所有线程重新检查
}

---

## **7. 总结**

- **`notify()`**：精准唤醒一个线程，适合**同质任务**（如单消费者）。
    
- **`notifyAll()`**：唤醒所有线程，适合**复杂条件协作**（如多条件等待）。
    
- **关键原则**：
    
    - 始终在 `while` 循环中调用 `wait()`。
        
    - 不确定时优先用 `notifyAll()`，避免线程饥饿。
        
    - 在明确场景下用 `notify()` 提升性能。
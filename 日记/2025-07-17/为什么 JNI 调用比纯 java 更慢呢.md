JNI 是 Java 和本地代码（如 C/C++）交互的桥梁，但它的调用开销显著高于纯 Java 方法调用，主要原因包括：

---

## **1. 跨语言边界开销（Java ↔ Native）**

### **(1) 数据转换（Marshaling/Unmarshaling）**

- **参数传递**：Java 对象（如 `String`、数组）需要转换为本地代码能理解的格式（如 C 的 `char*`、`jintArray`）。
    
- **返回值处理**：本地代码返回的结果需转换回 Java 对象。
    
- **示例**：
    
    java
    
    // Java → JNI → C
    native String process(String input); // Java 调用
    
    c
    
    // C 侧实现
    JNIEXPORT jstring JNICALL Java_MyClass_process(JNIEnv *env, jobject obj, jstring input) {
        const char *c_str = (*env)->GetStringUTFChars(env, input, NULL); // 转换 Java String → C 字符串
        char *result = process_in_c(c_str);                              // C 处理逻辑
        jstring java_result = (*env)->NewStringUTF(env, result);         // 转换 C 字符串 → Java String
        (*env)->ReleaseStringUTFChars(env, input, c_str);                // 释放资源
        return java_result;
    }
    
    **开销**：每次调用都涉及数据结构的深拷贝和内存分配。
    

### **(2) 线程状态切换**

- **Java 线程状态**：JNI 调用会触发线程从 **Java 模式** 切换到 **本地模式**，保存/恢复线程上下文（如寄存器状态）。
    
- **全局锁（JNI Critical Section）**：某些操作（如访问 Java 数组）需要获取 JVM 内部锁，可能引发竞争。
    

---

## **2. 失去 JVM 优化能力**

### **(1) 无法内联（Method Inlining）**

- 纯 Java 方法调用可能被 JIT 编译器内联，消除调用开销。
    
- JNI 调用是动态绑定的，JVM 无法静态分析其行为，故无法内联。
    

### **(2) 逃逸分析失效**

- JVM 的逃逸分析（Escape Analysis）可以优化对象分配（如栈上分配），但 JNI 可能强制对象逃逸到堆上。
    

### **(3) 无法进行即时编译（JIT）优化**

- 本地代码（如 C/C++）通常由预编译的二进制文件提供，JVM 无法对其应用运行时优化（如热点代码编译）。
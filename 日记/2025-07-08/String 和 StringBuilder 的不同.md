`String` 和 `StringBuilder` 在 Java 和 Kotlin 中都是用于处理字符串的类，但它们在**内存管理**、**性能**和**使用场景**上有显著区别。以下是它们的核心对比：

---

### **1. 不可变性 vs 可变性**

|**特性**|**`String`**|**`StringBuilder`**|
|---|---|---|
|**可变性**|**不可变**（创建后内容不可修改）|**可变**（可动态修改内容）|
|**内存开销**|每次修改会创建新对象，旧对象等待 GC 回收|直接修改内部字符数组，避免频繁内存分配|

**示例**：

java

// String 的不可变性
String s1 = "Hello";
s1 += " World"; // 创建新对象，原 "Hello" 成为垃圾

// StringBuilder 的可变性
StringBuilder sb = new StringBuilder("Hello");
sb.append(" World"); // 直接修改原对象

---

### **2. 性能对比**

|**操作**|**`String`**|**`StringBuilder`**|
|---|---|---|
|**单次拼接**|无显著差异|无显著差异|
|**循环拼接**|**极差**（O(n²) 时间）|**高效**（O(n) 时间）|

**循环拼接测试**（Java）：

java

// String（性能差）
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i; // 每次循环创建新对象
}

// StringBuilder（性能优）
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i); // 直接修改内部数组
}
String result = sb.toString();

---

### **3. 线程安全性**

|**特性**|**`String`**|**`StringBuilder`**|**`StringBuffer`**|
|---|---|---|---|
|**线程安全**|是（因不可变）|**否**|是（同步方法）|

**选择建议**：

- 单线程操作 → `StringBuilder`（更快）
    
- 多线程操作 → `StringBuffer`（线程安全但稍慢）
    

---

### **4. 底层实现**

|**类**|**存储结构**|**扩容机制**|
|---|---|---|
|`String`|final char[]（Java 8 前）  <br>byte[] + 编码标记（Java 9+）|不可变，每次修改创建新数组|
|`StringBuilder`|char[]（初始容量 16）|自动扩容（2n + 2）|

**`StringBuilder` 扩容逻辑**：

java

// 伪代码
void expandCapacity(int minCapacity) {
    int newCapacity = (value.length << 1) + 2; // 2倍 + 2
    if (newCapacity < minCapacity) {
        newCapacity = minCapacity;
    }
    value = Arrays.copyOf(value, newCapacity);
}

---

### **5. 使用场景建议**

|**场景**|**推荐类**|**原因**|
|---|---|---|
|静态字符串（如配置常量）|`String`|不可变性保证安全|
|单线程下频繁拼接/修改字符串|`StringBuilder`|避免内存浪费|
|多线程下字符串操作|`StringBuffer`|同步方法保证线程安全|
|SQL 拼接/JSON 构建|`StringBuilder`|减少临时对象生成|

---

### **6. Kotlin 的优化**

在 Kotlin 中：

- 字符串模板（`"$var"`）在编译后会自动优化为 `StringBuilder`。
    
- 使用 `buildString { }` 更简洁：
    
    kotlin
    
    val s = buildString {
        append("Hello")
        append(" ")
        append("World")
    }
    

---

### **总结**

- **`String`**：适合**不可变字符串**，操作简单但性能差。
    
- **`StringBuilder`**：适合**频繁修改字符串**，内存高效但非线程安全。
    
- **`StringBuffer`**：线程安全版 `StringBuilder`，性能略低。
    

**黄金法则**：  
在循环或高频修改字符串时，永远优先使用 `StringBuilder`！
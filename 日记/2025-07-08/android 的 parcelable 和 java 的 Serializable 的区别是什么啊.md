在 Android 开发中，`Parcelable` 和 `Serializable` 都是用于**对象序列化**的接口，但它们的实现方式、性能和使用场景有显著区别。以下是两者的详细对比：

---

### **1. 核心区别**

| **特性**    | **Parcelable** (Android)         | **Serializable** (Java) |
| --------- | -------------------------------- | ----------------------- |
| **所属平台**  | Android 专用                       | Java 标准库（跨平台）           |
| **序列化机制** | 手动实现，需覆写 `writeToParcel()` 等方法   | 自动序列化（反射实现）             |
| **性能**    | **高**（直接操作内存，无反射）                | **低**（反射生成临时对象，GC 压力大）  |
| **适用场景**  | Android 组件间传递数据（如 Intent、Bundle） | 网络传输或持久化存储（如文件、数据库）     |
| **代码复杂度** | 较高（需手动实现）                        | 极低（只需声明接口）              |
| **传输效率**  | 二进制格式，体积小                        | 默认使用 Java 序列化协议，体积较大    |
| **跨进程支持** | 支持（但需显式处理）                       | 支持                      |

---

### **2. 实现方式对比**

#### **(1) `Serializable` 的实现**

只需让类实现 `Serializable` 接口即可，无需额外代码：

java

// Java/Kotlin
public class User implements Serializable {
    private String name;
    private int age;
    // 自动序列化（反射完成）
}

#### **(2) `Parcelable` 的实现**

需手动实现 `Parcelable` 接口，包括：

- `writeToParcel()`：将对象字段写入 `Parcel`。
    
- `CREATOR`：从 `Parcel` 重建对象。
    

kotlin

// Kotlin 示例
class User(val name: String, val age: Int) : Parcelable {
    constructor(parcel: Parcel) : this(
        parcel.readString()!!,
        parcel.readInt()
    )

    override fun writeToParcel(parcel: Parcel, flags: Int) {
        parcel.writeString(name)
        parcel.writeInt(age)
    }

    companion object CREATOR : Parcelable.Creator<User> {
        override fun createFromParcel(parcel: Parcel): User {
            return User(parcel)
        }
        override fun newArray(size: Int): Array<User?> {
            return arrayOfNulls(size)
        }
    }

    override fun describeContents(): Int = 0
}

---

### **3. 性能差异**

#### **为什么 `Parcelable` 更快？**

1. **直接内存操作**  
    `Parcelable` 通过 `Parcel` 直接读写二进制数据，避免了反射开销。
    
2. **无临时对象**  
    `Serializable` 的反射机制会生成大量临时对象，增加 GC 负担。
    
3. **测试数据**  
    实测在 Android 中，`Parcelable` 的序列化速度比 `Serializable` **快 10 倍以上**。
    

---

### **4. 使用场景建议**

- **用 `Parcelable` 的场景**：
    
    - 在 Android 组件（如 Activity、Service）间传递数据。
        
    - 高频调用的数据序列化（如列表数据传递）。
        
- **用 `Serializable` 的场景**：
    
    - 需要跨平台存储或网络传输（如 JSON 转换）。
        
    - 快速原型开发（无需手动实现序列化）。
        

---

### **5. 进阶优化**

#### **(1) 使用 `@Parcelize` 注解（Kotlin）**

通过 Kotlin 扩展插件自动生成 `Parcelable` 代码：

kotlin

@Parcelize
data class User(val name: String, val age: Int) : Parcelable

需在 `build.gradle` 中启用：

gradle

android {
    androidExtensions {
        experimental = true
    }
}

#### **(2) 序列化工具替代方案**

- **JSON（Gson/Moshi）**：适合网络传输，但性能低于 `Parcelable`。
    
- **Protocol Buffers**：高性能二进制协议，适合复杂数据。
    

---

### **6. 总结**

- **优先用 `Parcelable`**：Android 内部数据传输的首选方案，性能极致。
    
- **慎用 `Serializable`**：仅在需要兼容 Java 标准库或快速开发时使用。
    
- **其他选择**：根据场景考虑 JSON 或 Protobuf 等跨平台方案。
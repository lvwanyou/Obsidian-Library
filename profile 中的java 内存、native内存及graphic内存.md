### **1. Java 内存（Java Heap）**

#### **存储内容**

- **Java/Kotlin 对象**：应用代码中创建的类实例（如 `Activity`、`String`、`ArrayList`）。
    
- **ART 虚拟机管理**：由 Android 的运行时（ART）自动分配和回收。
    

#### **关键特点**

- **垃圾回收（GC）**：当内存不足时，GC 会触发回收无引用的对象。
    
- **内存泄漏常见区域**：静态集合、未注销的监听器、长生命周期引用短生命周期对象（如 `Activity` 被单例持有）。
    

#### **监控指标**

- **Java Heap Size**：当前分配的堆大小。
    
- **Used/Free**：已使用和空闲内存。
    
- **内存快照（Heap Dump）**：分析对象引用链，定位泄漏。

### **2. Native 内存（Native Heap）**

#### **存储内容**

- **C/C++ 层分配的内存**：
    
    - 通过 `malloc()`、`new` 等直接分配。
        
    - JNI 调用的本地库（如 OpenCV、TensorFlow Lite）。
        
    - 系统组件（如 `Surface`、`MediaCodec`）的底层实现。
        

#### **关键特点**

- **手动管理**：需显式释放（`free()` 或析构函数），否则导致泄漏。
    
- **不受 GC 影响**：Java 的 GC 不会回收 Native 内存。
    

#### **监控指标**

- **Native Heap Size**：通过 `adb shell dumpsys meminfo <package>` 查看。
    
- **工具**：Android Studio 的 Native Memory Profiler 或 `jemalloc` 调试。


### **3. Graphics 内存（Graphics RAM）**

#### **存储内容**

- **GPU 相关资源**：
    
    - 纹理（Textures）、顶点缓冲区（VBO）、帧缓冲区（FBO）。
        
    - `Bitmap`、`Surface`、`OpenGL ES`/`Vulkan` 使用的显存。
        
- **共享内存**：部分数据可能在 CPU 和 GPU 间同步（如 `GraphicBuffer`）。
    

#### **关键特点**

- **高带宽占用**：大纹理或频繁上传数据会显著增加内存压力。
    
- **隐式分配**：`ImageView.setImageBitmap()` 或 `Canvas` 操作可能触发 GPU 内存分配。
#### **监控指标**

- **Graphics Allocation**：在 Android Profiler 的 "Memory" 选项卡中查看。
    
- **工具**：`adb shell dumpsys gfxinfo` 或 GPU 调试工具（如 RenderDoc）。
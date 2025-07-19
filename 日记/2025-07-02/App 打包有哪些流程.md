Android App 打包（构建 APK/AAB）的流程是一个多阶段的自动化过程，主要依赖 **Gradle** 和 **Android 构建工具链** 完成。以下是详细的流程解析和关键步骤说明：

---

### **1. 资源处理（Resource Processing）**

- **合并资源**：将模块（Module）中的 `res/`、`assets/` 和第三方库的资源合并。
    
- **资源编译**：
    
    - `XML` 文件（如布局、动画）被编译为二进制格式（减少体积）。
        
    - `PNG/JPG` 图片可能被优化（如 `WebP` 转换）。
        
    - 生成 `R.java` 文件（资源 ID 映射）。
        
- **资源链接**：生成 `resources.arsc`（资源索引表）。
    

**工具**：`aapt2` (Android Asset Packaging Tool 2)

---

### **2. 代码编译（Code Compilation）**

- **Java/Kotlin 编译**：
    
    - Kotlin 代码 → `.class` 文件（通过 `kotlinc`）。
        
    - Java 代码 → `.class` 文件（通过 `javac`）。
        
- **字节码优化**：
    
    - 使用 `R8` 或 `ProGuard` 进行代码混淆和优化（移除未使用的类/方法）。
        
    - 生成优化后的 `.dex` 文件（Dalvik 可执行格式）。
        

**工具**：`kotlinc`、`javac`、`R8/ProGuard`、`d8` (DEX 编译器)

---

### **3. 清单文件合并（Manifest Merging）**

- 合并所有模块的 `AndroidManifest.xml`，包括：
    
    - 权限声明。
        
    - 组件（Activity、Service 等）配置。
        
    - 元数据（如 `applicationId`、`versionCode`）。
        
- 处理冲突（如使用 `tools:replace` 覆盖库中的属性）。
    

**工具**：`manifest-merger`

---

### **4. 原生库处理（Native Libraries）**

- 如果包含 C/C++ 代码：
    
    - 编译为 `.so` 文件（通过 `NDK`）。
        
    - 按 ABI（如 `armeabi-v7a`、`arm64-v8a`）分类打包。
        

**工具**：`CMake`、`ndk-build`

---

### **5. 打包生成 APK/AAB**

#### **APK (Android Package Kit)**

1. **未签名的 APK**：
    
    - 将 `.dex`、资源、`lib/`、`AndroidManifest.xml` 等打包到 `.apk` 文件。
        
2. **签名 APK**：
    
    - 使用 `jarsigner` 或 `apksigner` 对 APK 进行签名（V1/V2/V3 签名方案）。
        
    - 生成最终的 `app-release.apk`。
        

#### **AAB (Android App Bundle)**

1. **生成动态特性模块**：
    
    - 按配置拆分资源、代码和原生库（如语言、屏幕密度）。
        
2. **签名**：
    
    - 使用 `bundletool` 生成签名的 `.aab` 文件。
        
3. **Google Play 优化**：
    
    - 用户下载时，Google Play 按设备生成最优化的 APK。
        

**工具**：`zipflinger` (APK)、`bundletool` (AAB)

---

### **6. 对齐优化（Zipalign）**

- **4 字节对齐**：优化 APK 内存映射效率，提升运行时性能。
    
- 仅对 APK 必需（AAB 不需要）。
    

**工具**：`zipalign`

---

### **7. 构建流程图解**

plaintext

源代码 (Java/Kotlin) → 编译 → .class → 优化 → .dex
资源文件 (res/, assets/) → 编译 → resources.arsc
AndroidManifest.xml → 合并 → 最终 Manifest
原生代码 (.cpp) → 编译 → .so
↓
合并为未签名 APK/AAB → 签名 → 对齐 (APK) → 最终包
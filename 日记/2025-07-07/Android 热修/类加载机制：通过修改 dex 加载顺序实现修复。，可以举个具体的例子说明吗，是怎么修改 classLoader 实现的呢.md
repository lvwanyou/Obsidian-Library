好的！我们以 **Tinker** 的热修复方案为例，详细说明如何通过 **修改 `ClassLoader` 的 `dexElements` 加载顺序** 来实现热修复。

---

## **1. Android 的类加载机制**
![[Pasted image 20250720124231.png]]
Android 使用 `PathClassLoader`（应用主类加载器）加载 APK 中的 `classes.dex`，其关键逻辑是：

1. 类加载时，`ClassLoader` 会遍历 `dexElements` 数组（存储所有 `dex` 文件）。
    
2. 按顺序查找类，**找到后立即返回，不再继续查找后续 `dex`**。
    

这意味着，如果我们把修复后的 `dex` **插入到 `dexElements` 的最前面**，系统就会优先加载修复后的类，而不是原类。

---

## **2. Tinker 的热修复实现步骤**

### **（1）生成补丁 `dex`**

- 修复代码后，编译生成一个 **仅包含修复类的新 `dex` 文件**（例如 `patch.dex`）。
    

### **（2）动态插入补丁 `dex`**

在应用启动时（如 `Application.attachBaseContext()`），通过反射修改 `PathClassLoader` 的 `dexElements`：
```java
// 1. 获取当前的 PathClassLoader
PathClassLoader pathClassLoader = (PathClassLoader) getClassLoader();

// 2. 反射获取 DexPathList 的 dexElements 字段
Class<?> dexPathListClass = Class.forName("dalvik.system.DexPathList");
Field dexElementsField = dexPathListClass.getDeclaredField("dexElements");
dexElementsField.setAccessible(true);

// 3. 获取当前的 dexElements 数组
Object dexPathList = getField(pathClassLoader, "pathList");
Object[] oldDexElements = (Object[]) dexElementsField.get(dexPathList);

// 4. 加载补丁 dex 文件，生成新的 Element 对象
File patchDexFile = new File("/sdcard/patch.dex");
Class<?> elementClass = Class.forName("dalvik.system.DexPathList$Element");
Method makeDexElements = dexPathListClass.getDeclaredMethod(
    "makePathElements", List.class, File.class, List.class);
List<File> patchFiles = new ArrayList<>();
patchFiles.add(patchDexFile);
Object[] newDexElements = (Object[]) makeDexElements.invoke(
    null, patchFiles, null, new ArrayList<>());

// 5. 合并新旧 dexElements（补丁 dex 放在最前面！）
Object[] finalDexElements = (Object[]) Array.newInstance(
    oldDexElements.getClass().getComponentType(),
    oldDexElements.length + newDexElements.length);
System.arraycopy(newDexElements, 0, finalDexElements, 0, newDexElements.length);
System.arraycopy(oldDexElements, 0, finalDexElements, newDexElements.length, oldDexElements.length);

// 6. 将合并后的 dexElements 设置回 ClassLoader
dexElementsField.set(dexPathList, finalDexElements);
```

### **（3）修复生效**

- 当应用访问某个类（如 `BuggyClass`）时：
    
    1. `ClassLoader` 先遍历 `finalDexElements`，**优先从补丁 `patch.dex` 中查找**。
        
    2. 如果找到修复后的类，直接返回；否则继续查找原 APK 中的 `classes.dex`。
        

---

## **3. 关键点解析**

### **（1）为什么补丁 `dex` 要放在前面？**

- `dexElements` 是数组结构，类加载时按顺序查找，**第一个匹配的类会被使用**。
    
- 将补丁 `dex` 插到前面，确保修复类优先加载。
    

### **（2）如何避免 `CLASS_ISPREVERIFIED` 问题？**

在 Android 5.0 之前，类加载时会进行校验（`verify`），如果发现某个类的直接引用类不在同一个 `dex`，会抛出异常。  
**Tinker 的解决方案**：

- 在编译阶段，**对所有类插入一个来自独立 `dex` 的空引用**，避免原类被打上 `CLASS_ISPREVERIFIED` 标记。
    
- 例如：
```java
public class BuggyClass {
        static {
            // 引用一个单独 dex 中的类，避免验证
            System.out.println(AntiVerifyClass.class);
        }
    }
```

### **（3）为什么需要重启应用？**

- **类加载后无法卸载**：Android 的 `ClassLoader` 不支持卸载已加载的类。
    
- **重启后重新加载**：重启应用时，`ClassLoader` 会重新初始化，从而加载新的 `dexElements` 顺序。
    

---

## **4. 对比其他方案**

|方案|修改方式|是否需要重启|兼容性|
|---|---|---|---|
|**Tinker（类加载）**|反射修改 `dexElements`|是|高（全版本）|
|**AndFix（Native）**|替换 `ArtMethod` 指针|否|中（机型适配）|
|**Robust（插桩）**|编译时生成代理类，运行时切换|部分情况|高|

---

## **5. 总结**

- **Tinker 的热修复核心**：  
    通过反射将补丁 `dex` 插入 `ClassLoader` 的 `dexElements` 数组最前面，让修复类优先加载。
    
- **关键步骤**：
    
    1. 生成补丁 `dex` → 2. 反射修改 `dexElements` → 3. 重启生效。
        
- **优势**：兼容性好，适合大规模应用；**劣势**：需要重启。
    

这种方案虽然依赖反射黑科技，但在主流热修复框架中稳定性最高，也是微信、QQ 等大型应用的选择。
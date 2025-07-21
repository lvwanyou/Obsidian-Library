在 Android 中，**Window（窗口）** 是一个核心概念，它代表了一个可视化的界面单元，负责管理 **UI 的绘制、事件分发、层级控制** 等。以下是它的关键点解析：

---

### **1. Window 的本质**

- **Window 是抽象概念**，没有实际可见的形态，但它是所有视图（`View`）的容器。
    
- 每个 **Activity、Dialog、Toast、PopupWindow** 都对应一个独立的 Window。
    
- **Window 的背后是 `ViewTree`**：它通过 `DecorView`（根视图）管理整个 UI 层级。
    

---

### **2. Window 的核心组成**

|组件|作用|
|---|---|
|**WindowManager**|负责 Window 的添加、删除、更新（通过 `WindowManager.addView()` 等）。|
|**DecorView**|根视图，包含系统装饰（如状态栏、导航栏）和用户自定义的 `ContentView`。|
|**PhoneWindow**|Window 的唯一实现类（Android 中 `Window` 是抽象类，由 `PhoneWindow` 实现）。|
|**Surface**|底层绘图表面，Window 的内容最终通过 `Surface` 渲染到屏幕上。|

---

### **3. Window 的工作流程**

#### **(1) Window 的创建（以 Activity 为例）**

```java
// ActivityThread.handleResumeActivity()
final WindowManager.LayoutParams l = new WindowManager.LayoutParams();
wm.addView(decorView, l); // 将 DecorView 添加到 WindowManager
```

- Activity 启动时，会创建一个 `PhoneWindow`，并关联 `DecorView`。
    
- 通过 `WindowManager` 将 `DecorView` 添加到屏幕。
    

#### **(2) 视图的层级关系**

text

Window
└── DecorView
    ├── TitleBar（可选）
    └── ContentView（setContentView() 设置的布局）

#### **(3) 事件分发流程**

text

InputEvent → Window → DecorView → ViewGroup/View

- 触摸事件先由 `Window` 接收，再传递给 `DecorView`，最后分发到具体的 `View`。
    

---

### **4. Window 的类型（Type）**

Window 有不同的类型，决定其显示层级和特性（通过 `WindowManager.LayoutParams.type` 设置）：

|类型|值|用途|
|---|---|---|
|**应用窗口**|1~99|Activity 的默认窗口，位于应用层级。|
|**子窗口**|1000~1999|必须依附于父窗口（如 Dialog）。|
|**系统窗口**|2000~2999|无需 Activity 承载（如 Toast、状态栏、输入法窗口）。|

**示例**：

```java
// 系统级悬浮窗
WindowManager.LayoutParams params = new WindowManager.LayoutParams(
    WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY // 系统窗口类型
);
```

---

### **5. Window 与 Surface 的关系**

- **Window 是逻辑单元**：管理视图层级和事件。
    
- **Surface 是物理单元**：由 `SurfaceFlinger` 管理，负责最终像素渲染。
    
- **联系**：每个 Window 对应一个 `Surface`，`ViewRootImpl` 通过 `Surface` 将 UI 绘制到屏幕上。
    

---

### **6. 常见问题**

#### **Q1: Dialog 和 PopupWindow 的 Window 有什么区别？**

- **Dialog**：使用 `TYPE_APPLICATION` 类型，必须依附于 Activity。
    
- **PopupWindow**：默认使用 `TYPE_APPLICATION_PANEL`（子窗口），可通过 `TYPE_APPLICATION_OVERLAY` 变成系统级悬浮窗。
    

#### **Q2: 如何获取当前 Activity 的 Window？**

```java
Activity activity = ...;
Window window = activity.getWindow();
View decorView = window.getDecorView();
```

#### **Q3: 为什么 SurfaceView 能在非 UI 线程绘制？**

- `SurfaceView` 拥有独立的 `Surface`，不依赖主线程的 `View` 绘制流程。
    

---

### **7. 总结**

- **Window 是 Android UI 的骨架**：承载视图层级，管理显示和交互。
    
- **关键角色**：`WindowManager`、`DecorView`、`PhoneWindow`、`Surface`。
    
- **控制层级**：通过 `WindowManager.LayoutParams` 可设置窗口类型、透明度、位置等。
    

理解 Window 机制，能帮助你更深入掌握 **UI 渲染、事件分发、悬浮窗开发** 等高级场景。
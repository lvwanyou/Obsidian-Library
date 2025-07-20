在 Android 的渲染体系中，`TextureView` 的渲染流程既涉及 **UI 线程** 的 `DisplayList` 生成，又依赖 **`RenderThread`** 的异步渲染，但其机制与普通 `View` 或 `SurfaceView` 有所不同。以下是详细解析：

---

### **1. `TextureView` 的渲染流程**

`TextureView` 的渲染分为两个关键阶段，**需要 UI 线程和 `RenderThread` 协作完成**：

#### **(1) UI 线程：`DisplayList` 生成与更新**

- **`onDraw` 阶段**：  
    `TextureView` 作为一个普通 `View`，其 `onDraw()` 方法会在 UI 线程被调用，生成 `DisplayList` 指令。但这些指令**不直接绘制内容**，而是描述如何渲染其内部的 `SurfaceTexture` 纹理。
    
    java
    
    @Override
    protected void onDraw(Canvas canvas) {
        // 硬件加速下，生成 DisplayList 指令（如 "DrawTextureOp"）
        canvas.drawTexture(mSurfaceTexture, ...); // 将 SurfaceTexture 绑定到纹理
    }
    
- **`SurfaceTexture` 更新**：  
    `TextureView` 通过 `SurfaceTexture` 接收外部内容（如相机预览、视频解码），当新帧到达时，UI 线程会触发 `invalidate()`，重新生成 `DisplayList`。
    

#### **(2) `RenderThread`：纹理合成与渲染**

- **纹理上传**：  
    `SurfaceTexture` 的内容（如相机帧）由生产者（如 `MediaCodec`）直接写入到 GPU 纹理，**不经过 UI 线程**。
    
- **合成渲染**：  
    `RenderThread` 将 `TextureView` 的 `DisplayList`（包含纹理引用）与其他 `View` 的 `DisplayList` 合并，最终通过 GPU 合成到屏幕。
    

---

### **2. 与 `SurfaceView` 的对比**

|**特性**|`TextureView`|`SurfaceView`|
|---|---|---|
|**渲染线程**|UI 线程 + `RenderThread`|直接由 `RenderThread` 或专用线程处理|
|**`DisplayList` 生成**|是（描述纹理如何绘制）|否（绕过 View 系统）|
|**合成层级**|与普通 `View` 统一合成|独立 `Surface`，由 `SurfaceFlinger` 处理|
|**透明度支持**|天然支持（通过 `DisplayList` 混合）|需手动设置透明像素格式|

---

### **3. 关键结论**

- **`TextureView` 的渲染路径**：  
    **需要 UI 线程生成 `DisplayList`**（定义纹理如何绘制），但纹理内容的生产（如视频解码）和最终合成由 `RenderThread` 异步完成。
    
- **性能权衡**：
    
    - **优点**：支持动画、变换和透明度，与普通 `View` 兼容性好。
        
    - **缺点**：比 `SurfaceView` 多一次 `DisplayList` 生成和纹理拷贝，性能略低。
        

---

### **4. 流程图示**

plaintext

┌───────────────────────┐    ┌───────────────────────┐
│     生产者（如相机）     │    │       UI 线程         │
│                       │    │                       │
│ 写入帧到 SurfaceTexture├───►│ onDraw → DisplayList  │
└──────────┬────────────┘    └──────────┬────────────┘
           │                            │
           │                            ▼
           │                 ┌───────────────────────┐
           │                 │     RenderThread      │
           └────────────────►│                       │
                             │ 合成纹理到屏幕        │
                             └───────────────────────┘

---

### **5. 开发者注意事项**

1. **避免 UI 线程阻塞**：  
    `TextureView` 依赖 UI 线程生成 `DisplayList`，若主线程卡顿会导致渲染延迟（表现为掉帧）。
    
2. **纹理更新回调**：  
    通过 `SurfaceTexture.OnFrameAvailableListener` 监听新帧，及时调用 `invalidate()`：
    
    java
    
    mSurfaceTexture.setOnFrameAvailableListener(surfaceTexture -> {
        textureView.postInvalidate(); // 请求重绘
    });
    
3. **性能优化**：
    
    - 对高频更新场景（如 60fps 视频），优先考虑 `SurfaceView`。
        
    - 减少 `TextureView` 的复杂变换（如矩阵运算）。
        

---

### **6. 底层原理验证**

- **方法 1：检查 `Canvas` 类型**  
    在 `TextureView` 的 `onDraw` 中打印 `Canvas` 类型：
    
    java
    
    @Override
    protected void onDraw(Canvas canvas) {
        Log.d("TextureView", "Canvas class: " + canvas.getClass().getName());
        // 输出: android.view.DisplayListCanvas（硬件加速）
    }
    
- **方法 2：Systrace 分析**  
    使用 `systrace` 工具可观察到 `RenderThread` 和 UI 线程的协作过程。
    

---

总结来说，`TextureView` **需要 UI 线程生成 `DisplayList`** 来描述纹理的绘制方式，但实际的纹理数据和最终合成由 `RenderThread` 处理。这种设计使其在灵活性和性能之间取得了平衡。




## TextureView 需要 UI 线程生成 DisplayList 来描述纹理的绘制方式 ，这个怎么理解呢
在 Android 的渲染体系中，`TextureView` 需要 **UI 线程生成 `DisplayList`** 来描述其纹理的绘制方式，这一过程的核心在于 `TextureView` 的混合渲染机制。以下是逐步拆解：

---

### **1. `TextureView` 的本质**

`TextureView` 是一个 **普通 `View` 的子类**，但它通过 `SurfaceTexture` 将外部内容（如相机预览、视频解码）绑定到 OpenGL 纹理上。它的特殊之处在于：

- **内容来源**：由独立的生产者（如 `MediaCodec`、相机）直接写入 `SurfaceTexture` 的纹理（不经过 UI 线程）。
    
- **绘制方式**：需要通过 UI 线程的 `onDraw` 生成 `DisplayList`，告诉系统“如何绘制这块纹理”。
    

---

### **2. 为什么需要 UI 线程生成 `DisplayList`？**

虽然纹理内容由其他线程生产，但 `TextureView` 的 **位置、大小、透明度、变换矩阵** 等属性仍需由 UI 线程管理。具体来说：

1. **`DisplayList` 的作用**：
    
    - 记录 `TextureView` 的 **绘制指令**（如 `drawTexture`），而非实际像素数据。
        
    - 包含纹理的 **变换矩阵**（平移/旋转/缩放）、**透明度**、**裁剪区域** 等信息。
        
2. **UI 线程的职责**：
    
    - 计算 `TextureView` 的布局（`measure/layout`）。
        
    - 在 `onDraw` 中生成 `DisplayList`，描述如何将纹理合成到屏幕上。
        

---

### **3. 具体流程分解**

#### **(1) 生产者填充纹理（非 UI 线程）**

- 例如相机通过 `SurfaceTexture` 将预览帧写入 GPU 纹理：
    
    java
    
    // 生产者（如相机）直接操作纹理，不阻塞 UI 线程
    camera.setPreviewTexture(textureView.surfaceTexture);
    

#### **(2) UI 线程生成 `DisplayList`**

- `TextureView` 的 `onDraw` 生成绘制指令：
    
    java
    
    @Override
    protected void onDraw(Canvas canvas) {
        // 硬件加速下，canvas 是 DisplayListCanvas
        canvas.drawTexture(mSurfaceTexture, 0, 0, mWidth, mHeight, mPaint);
        // ↑ 生成一条 "DrawTextureOp" 指令，记录到 DisplayList
    }
    
    - **关键点**：  
        `drawTexture` 并不实际绘制像素，而是将 `SurfaceTexture` 的纹理 ID 和绘制参数（如矩阵）保存到 `DisplayList`。
        

#### **(3) `RenderThread` 执行合成**

- `RenderThread` 读取 `DisplayList`，结合纹理内容进行最终渲染：
    
    plaintext
    
    DisplayList 指令示例：
    1. 绑定纹理 ID: 1234
    2. 应用变换矩阵: [平移X, 缩放Y, ...]
    3. 合成到屏幕坐标 (0,0)-(100,100)
    

---

### **4. 类比理解**

将 `TextureView` 的渲染比作 **贴海报**：

- **生产者**：另一个人（非 UI 线程）负责准备海报内容（纹理数据）。
    
- **UI 线程**：你（UI 线程）决定海报贴在哪里、多大角度、是否半透明（生成 `DisplayList`）。
    
- **`RenderThread`**：工人（`RenderThread`）根据你的指示，把海报贴到墙上（屏幕）。
    

---

### **5. 与 `SurfaceView` 的差异**

|**步骤**|`TextureView`|`SurfaceView`|
|---|---|---|
|**内容生产**|生产者直接写纹理（非 UI 线程）|生产者直接写 `Surface`（非 UI 线程）|
|**绘制指令生成**|UI 线程生成 `DisplayList`|无 `DisplayList`（绕过 View 系统）|
|**合成方式**|由 `RenderThread` 与其他 `View` 统一合成|由 `SurfaceFlinger` 独立合成|

---

### **6. 性能影响**

- **优势**：
    
    - 支持与普通 `View` 相同的属性（如动画、透明度）。
        
    - 纹理更新不阻塞 UI 线程（生产者直接操作 GPU 纹理）。
        
- **劣势**：
    
    - 比 `SurfaceView` 多一次 `DisplayList` 生成和纹理混合，性能略低。
        
    - UI 线程的 `onDraw` 可能成为瓶颈（若频繁触发）。
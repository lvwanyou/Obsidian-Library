在 Android 的视图系统中，`postInvalidate()` 和 `requestLayout()` 都是用于触发视图更新的方法，但它们的**作用范围**、**触发流程**和**使用场景**有本质区别。以下是详细对比：

---

### **1. 核心区别总结**

|**特性**|`postInvalidate()`|`requestLayout()`|
|---|---|---|
|**作用目标**|仅触发 `onDraw()` 重绘|触发 `measure()` + `layout()` + `onDraw()`|
|**线程安全性**|可在非 UI 线程调用|必须在 UI 线程调用|
|**性能开销**|较低（仅重绘）|较高（重新测量和布局）|
|**典型场景**|动态更新视图内容（如动画、数据变化）|视图尺寸或结构变化（如文本长度改变）|

---

### **2. 工作流程对比**

#### **(1) `postInvalidate()` 的流程**

plaintext

1. 调用 `postInvalidate()` → 向 UI 线程的消息队列发送重绘请求。
2. UI 线程处理消息时，调用目标 View 的 `invalidate()`。
3. 系统标记脏区域（Dirty Rect），触发 `onDraw()`。
4. `RenderThread` 仅重绘该区域（不触发布局和测量）。

**特点**：

- 仅重绘内容，不改变视图大小或位置。
    
- 适合频繁调用的场景（如动画）。
    

#### **(2) `requestLayout()` 的流程**

plaintext

1. 调用 `requestLayout()` → 向上回溯到 ViewRootImpl。
2. 触发全局 `measure()` 和 `layout()` 遍历（从根视图开始）。
3. 如果视图尺寸或位置变化，再触发 `onDraw()`。
4. `RenderThread` 处理新的绘制指令。

**特点**：

- 重新计算整个视图树的尺寸和位置。
    
- 开销大，需谨慎使用。
    

---

### **3. 使用场景示例**

#### **(1) `postInvalidate()` 适用场景**

- **动态绘制内容**（如自定义 View 的实时更新）：
    
    java
    
    class CircleView extends View {
        private float radius = 10;
    
        void expand() {
            radius += 5;
            postInvalidate(); // 仅触发 onDraw()，不重新测量
        }
    
        @Override
        protected void onDraw(Canvas canvas) {
            canvas.drawCircle(100, 100, radius, paint);
        }
    }
    

#### **(2) `requestLayout()` 适用场景**

- **视图尺寸变化**（如 TextView 文本长度改变）：
    
    java
    
    TextView textView = findViewById(R.id.textView);
    textView.setText("New Long Text"); // 文本变长可能导致宽度不足
    textView.requestLayout(); // 触发重新测量和布局
    

---

### **4. 关键注意事项**

#### **(1) 线程安全性**

- `postInvalidate()`：  
    内部通过 `Handler` 将重绘请求抛到 UI 线程，**可在子线程调用**。
    
    java
    
    new Thread(() -> {
        view.postInvalidate(); // 安全
    }).start();
    
- `requestLayout()`：  
    必须直接在 **UI 线程调用**，否则会抛出异常：
    
    java
    
    new Thread(() -> {
        runOnUiThread(() -> view.requestLayout()); // 必须切换线程
    }).start();
    

#### **(2) 性能优化**

- 避免在 `onDraw()` 中调用 `requestLayout()`，可能导致递归布局和卡顿。
    
- 对高频更新（如动画），优先用 `postInvalidate()` 而非 `invalidate()`（后者需在 UI 线程调用）。
    

#### **(3) 组合使用案例**

若视图内容变化且可能影响尺寸（如图片加载后宽高改变）：

java

void loadImage(Bitmap bitmap) {
    imageView.setImageBitmap(bitmap);
    if (bitmap.getWidth() != imageView.getWidth()) {
        imageView.requestLayout(); // 尺寸变化，需要重新布局
    } else {
        imageView.postInvalidate(); // 仅内容变化，重绘即可
    }
}

---

### **5. 底层原理**

- **`invalidate()` vs `postInvalidate()`**：  
    两者最终都标记脏区域，但 `postInvalidate()` 通过 `ViewRootImpl` 的 `Handler` 实现线程切换。
    
- **`requestLayout()` 的向上回溯**：  
    调用后会从当前 View 向上遍历到 `ViewRootImpl`，触发全局的 `performMeasure()` 和 `performLayout()`。
    

---

### **6. 常见误区**

- **误用 `requestLayout()`**：  
    在仅需重绘时调用它，会导致不必要的测量/布局计算（如旋转动画中视图尺寸未变）。
    
- **忽略线程限制**：  
    在子线程直接调用 `requestLayout()` 会引发 `CalledFromWrongThreadException`。
    

---

### **总结**

- **用 `postInvalidate()`**：当视图的**内容变化但尺寸不变**时（如动画、数据更新）。
    
- **用 `requestLayout()`**：当视图的**尺寸或位置可能变化**时（如文本、图片尺寸改变）。
    
- **线程选择**：子线程更新 UI 时，优先用 `postInvalidate()` 或切换到 UI 线程调用 `requestLayout()`。
在 Android 中，**属性动画（Property Animation）是否会导致 `onMeasure` 和 `onLayout` 重新执行，取决于动画修改的属性类型**。以下是详细分析：

---

### **1. 属性动画对 `onMeasure` 和 `onLayout` 的影响**

#### **(1) 不会触发 `onMeasure` 和 `onLayout` 的情况**

如果属性动画修改的是 **仅影响绘制（`onDraw`）的属性**，例如：

- `alpha`（透明度）
    
- `rotation`（旋转）
    
- `scaleX`/`scaleY`（缩放）
    
- `translationX`/`translationY`（平移）
    

这些属性的变化**不会改变 View 的实际尺寸或位置**，因此不会触发 `onMeasure` 和 `onLayout`，仅会通过 `invalidate()` 触发 `onDraw` 重绘56。

#### **(2) 会触发 `onMeasure` 和 `onLayout` 的情况**

如果属性动画修改的是 **影响布局（Layout）的属性**，例如：

- `width`/`height`（直接修改 View 的尺寸）
    
- `layoutParams`（如 `layout_width` 或 `layout_height`）
    

这些变化会影响 View 的测量和布局，因此会触发 `requestLayout()`，导致 `onMeasure` 和 `onLayout` 重新执行46。

---

### **2. 底层机制**

- **`invalidate()`**：仅标记需要重绘（`onDraw`），不触发 `onMeasure` 和 `onLayout`。
    
- **`requestLayout()`**：标记需要重新测量和布局，会依次调用 `onMeasure` 和 `onLayout`，并最终触发 `onDraw`4。
    

属性动画默认使用 `invalidate()` 来更新 UI，但如果修改的是布局相关属性（如 `width`），则会调用 `requestLayout()`6。

---

### **3. 优化建议**

- **避免频繁触发 `requestLayout`**：如果动画仅涉及视觉变化（如平移、旋转），优先使用 `translationX/Y` 或 `scaleX/Y`，而不是直接修改 `layoutParams`。
    
- **硬件加速（Hardware Layer）**：对于复杂动画，可启用硬件层（`setLayerType(LAYER_TYPE_HARDWARE, null)`），减少 `onDraw` 的计算开销6。
    

---

### **4. 示例对比**

#### **不会触发 `onMeasure`/`onLayout` 的动画**

java

// 仅触发 onDraw
ObjectAnimator.ofFloat(view, "translationX", 0f, 100f).start();

#### **会触发 `onMeasure`/`onLayout` 的动画**

java

// 修改宽度，触发 requestLayout
ObjectAnimator.ofInt(view, "width", 100, 200).start();

---

### **5. 总结**

|**动画属性**|**是否触发 `onMeasure`/`onLayout`**|**原因**|
|---|---|---|
|`alpha`/`rotation`|❌ 否|仅影响绘制，不改变布局。|
|`translationX/Y`|❌ 否|视觉偏移，不影响实际位置。|
|`scaleX/Y`|❌ 否|缩放绘制内容，不改变尺寸。|
|`width`/`height`|✅ 是|直接修改 View 的测量尺寸。|
|`layoutParams`|✅ 是|影响父布局的排列计算。|

**结论**：  
属性动画默认不会导致 `onMeasure` 和 `onLayout` 重新执行，除非动画直接修改了影响布局的属性（如 `width` 或 `layoutParams`）。合理选择动画属性可优化性能
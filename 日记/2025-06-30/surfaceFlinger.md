以下是你代码中的操作与 SurfaceFlinger 的交互关系：

|**你的代码**|**底层发生的事件**|**SurfaceFlinger 的参与**|
|---|---|---|
|`surfaceHolder.lockCanvas()`|从 `BufferQueue` 申请一块 `GraphicBuffer`，返回对应的 `Canvas` 对象。|监控 `BufferQueue` 状态，确保缓冲区可用。|
|`canvas.drawBitmap(bitmap, x, y)`|将 `Bitmap` 数据写入 `GraphicBuffer` 的共享内存。|无直接交互（纯内存操作）。|
|`surfaceHolder.unlockCanvasAndPost()`|将 `GraphicBuffer` 放回 `BufferQueue`，并通知 SurfaceFlinger 有新帧待处理。|收到通知后，在下一个 VSYNC 周期取出缓冲区，与其他 `Layer` 合成（如状态栏、导航栏）。|
|**屏幕刷新**|显示驱动从帧缓冲区读取数据。|
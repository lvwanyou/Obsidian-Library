在 Android 中使用 `MediaCodec` 解码视频并通过 `GLSurfaceView` 渲染到屏幕，是一个高效的硬件加速方案，尤其适合处理高分辨率或高帧率视频。以下是完整的流程解析和代码实现：

---

### **1. 核心流程概述**

1. **初始化 `MediaCodec`**：配置为解码器，绑定视频数据源（如文件或网络流）。
    
2. **创建 `SurfaceTexture`**：作为 `MediaCodec` 的输出表面，接收解码后的帧。
    
3. **绑定到 `GLSurfaceView`**：将 `SurfaceTexture` 转为 OpenGL 纹理，在 `GLSurfaceView.Renderer` 中绘制。
    
4. **实时渲染**：解码器推送帧 → `SurfaceTexture` 更新纹理 → OpenGL 渲染到屏幕。
    

---

### **2. 详细步骤与代码实现**

#### **(1) 初始化 `MediaCodec` 解码器**
```java
// 1. 创建 MediaExtractor 获取视频格式（如编码格式、分辨率）
MediaExtractor extractor = new MediaExtractor();
extractor.setDataSource(videoPath);
int videoTrackIndex = selectVideoTrack(extractor); // 选择视频轨道
extractor.selectTrack(videoTrackIndex);
MediaFormat format = extractor.getTrackFormat(videoTrackIndex);

// 2. 创建解码器（硬解优先）
MediaCodec codec = MediaCodec.createDecoderByType(format.getString(MediaFormat.KEY_MIME));
codec.configure(format, surface, null, 0); // 关键：绑定到 Surface
codec.start();
```

#### **(2) 创建 `SurfaceTexture` 并绑定到 OpenGL 纹理**
```java
// 1. 生成 OpenGL 纹理 ID
int[] textureId = new int[1];
GLES20.glGenTextures(1, textureId, 0);
GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, textureId[0]);
GLES20.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);

// 2. 创建 SurfaceTexture 并监听帧更新
SurfaceTexture surfaceTexture = new SurfaceTexture(textureId[0]);
surfaceTexture.setOnFrameAvailableListener(new SurfaceTexture.OnFrameAvailableListener() {
    @Override
    public void onFrameAvailable(SurfaceTexture surfaceTexture) {
        glSurfaceView.requestRender(); // 触发 GLSurfaceView 重绘
    }
});

// 3. 将 SurfaceTexture 包装为 Surface，供 MediaCodec 输出
Surface surface = new Surface(surfaceTexture);
```

#### **(3) 配置 `GLSurfaceView` 渲染器**
```java
glSurfaceView.setEGLContextClientVersion(2);
glSurfaceView.setRenderer(new GLSurfaceView.Renderer() {
    private float[] mtx = new float[16];

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        // 初始化 OpenGL 环境
        GLES20.glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        // 1. 更新纹理（获取最新视频帧）
        surfaceTexture.updateTexImage();
        surfaceTexture.getTransformMatrix(mtx);

        // 2. 渲染纹理到屏幕
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
        renderTexture(textureId[0], mtx); // 自定义绘制方法
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0, 0, width, height);
    }
});

```

#### **(4) 解码循环（在子线程执行）**
```java
ByteBuffer[] inputBuffers = codec.getInputBuffers();
MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
boolean isEOS = false;

while (!isEOS) {
    // 1. 将视频数据送入解码器
    int inputBufferIndex = codec.dequeueInputBuffer(10000);
    if (inputBufferIndex >= 0) {
        ByteBuffer inputBuffer = inputBuffers[inputBufferIndex];
        int sampleSize = extractor.readSampleData(inputBuffer, 0);
        if (sampleSize < 0) {
            codec.queueInputBuffer(inputBufferIndex, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM);
            isEOS = true;
        } else {
            codec.queueInputBuffer(inputBufferIndex, 0, sampleSize, extractor.getSampleTime(), 0);
            extractor.advance();
        }
    }

    // 2. 获取解码后的帧（自动渲染到 Surface）
    int outputBufferIndex = codec.dequeueOutputBuffer(bufferInfo, 10000);
    if (outputBufferIndex >= 0) {
        codec.releaseOutputBuffer(outputBufferIndex, true); // true 表示渲染到 Surface
    }
}
```

---

### **3. 关键点解析**

#### **(1) 为什么用 `GL_TEXTURE_EXTERNAL_OES`？**

- 视频解码器输出的帧格式通常是 **YUV**，而 `SurfaceTexture` 通过扩展纹理类型 `GL_TEXTURE_EXTERNAL_OES` 直接支持此类数据，避免手动转换。
    

#### **(2) 性能优化**

- **零拷贝渲染**：`MediaCodec` 输出直接绑定到 `SurfaceTexture`，GPU 直接处理数据，无需 CPU 参与。
    
- **硬件加速**：解码（`MediaCodec`）和渲染（OpenGL）均通过硬件实现。
    

#### **(3) 内存管理**

- 及时释放资源：
```java
    codec.stop();
    codec.release();
    surface.release();
    surfaceTexture.release();
```

### **总结**

- **`MediaCodec`**：负责硬解视频，输出到 `Surface`。
    
- **`SurfaceTexture`**：桥接解码器和 OpenGL，将帧转为纹理。
    
- **`GLSurfaceView`**：提供渲染环境和线程，高效绘制纹理。
    

这种方案充分利用 Android 的硬件加速能力，适合高性能视频播放、实时滤镜等场景。如需处理音频或更复杂的同步逻辑，可结合 `MediaSync` 或自定义音视频同步机制。
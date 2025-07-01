1. 通过 canvas 渲染
```
public class MainActivity extends AppCompatActivity implements SurfaceHolder.Callback {

    private SurfaceView surfaceView;
    private SurfaceHolder surfaceHolder;
    private Bitmap bitmap;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 初始化 SurfaceView
        surfaceView = findViewById(R.id.surface_view);
        surfaceHolder = surfaceView.getHolder();
        surfaceHolder.addCallback(this);

        // 创建一个测试 Bitmap（例如红色矩形）
        bitmap = Bitmap.createBitmap(400, 400, Bitmap.Config.ARGB_8888);
        Canvas bitmapCanvas = new Canvas(bitmap);
        Paint paint = new Paint();
        paint.setColor(Color.RED);
        bitmapCanvas.drawRect(0, 0, 400, 400, paint);
    }

    // --- SurfaceHolder.Callback 接口方法 ---
    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        // Surface 已创建，开始渲染
        drawBitmapToSurface();
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
        // Surface 尺寸变化时重新渲染
        drawBitmapToSurface();
    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        // Surface 被销毁时释放资源
    }

    // --- 核心渲染方法 ---
    private void drawBitmapToSurface() {
        Canvas surfaceCanvas = null;
        try {
            // 1. 锁定 Surface 的 Canvas
            surfaceCanvas = surfaceHolder.lockCanvas();

            // 2. 清空画布（可选）
            surfaceCanvas.drawColor(Color.BLACK);

            // 3. 将 Bitmap 绘制到 Surface 的 Canvas 上
            if (bitmap != null) {
                // 居中绘制 Bitmap
                float left = (surfaceCanvas.getWidth() - bitmap.getWidth()) / 2f;
                float top = (surfaceCanvas.getHeight() - bitmap.getHeight()) / 2f;
                surfaceCanvas.drawBitmap(bitmap, left, top, null);
            }

        } finally {
            // 4. 提交绘制内容到 SurfaceFlinger
            if (surfaceCanvas != null) {
                surfaceHolder.unlockCanvasAndPost(surfaceCanvas);
            }
        }
    }
}

```

```
TextureView textureView = findViewById(R.id.texture_view);
textureView.setSurfaceTextureListener(new TextureView.SurfaceTextureListener() {
    @Override
    public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height) {
        Canvas canvas = textureView.lockCanvas();
        canvas.drawBitmap(bitmap, 0, 0, null);
        textureView.unlockCanvasAndPost(canvas);
    }
    // ... 其他回调方法
});
```



GlSurfaceView 做渲染
- 通过 `GLSurfaceView` 和 OpenGL ES 2.0 渲染 Bitmap 的完整流程包括：
    
    1. **初始化 EGL 环境**（由 `GLSurfaceView` 自动完成）。
        
    2. **加载纹理**（将 Bitmap 上传到 GPU）。
        
    3. **编写着色器**（定义渲染规则）。
        
    4. **绘制帧**（绑定数据并调用绘制指令）。
        
- 最终由 SurfaceFlinger 在 VSYNC 信号触发时合成上屏。
```
public class BitmapRenderer implements GLSurfaceView.Renderer {
    private final Context context;
    private int textureId;
    private final FloatBuffer vertexBuffer;
    private final FloatBuffer textureBuffer;
    private int program;

    // 顶点坐标（NDC坐标系，x/y范围[-1,1]）
    private static final float[] VERTEX_COORDS = {
        -1f, -1f,  // 左下
        1f, -1f,   // 右下
        -1f, 1f,   // 左上
        1f, 1f     // 右上
    };

    // 纹理坐标（UV坐标系，范围[0,1]）
    private static final float[] TEXTURE_COORDS = {
        0f, 1f,    // 左下
        1f, 1f,    // 右下
        0f, 0f,    // 左上
        1f, 0f     // 右上
    };

    public BitmapRenderer(Context context) {
        this.context = context;
        // 初始化顶点和纹理坐标缓冲区
        vertexBuffer = ByteBuffer.allocateDirect(VERTEX_COORDS.length * 4)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer()
                .put(VERTEX_COORDS);
        vertexBuffer.position(0);

        textureBuffer = ByteBuffer.allocateDirect(TEXTURE_COORDS.length * 4)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer()
                .put(TEXTURE_COORDS);
        textureBuffer.position(0);
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        // 初始化 OpenGL ES 环境
        GLES20.glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
        // 加载纹理
        textureId = loadTexture(R.drawable.test_image);
        // 编译着色器程序
        program = createProgram();
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0, 0, width, height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
        // 绑定纹理和着色器程序
        GLES20.glUseProgram(program);
        GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureId);

        // 传递顶点坐标
        int positionHandle = GLES20.glGetAttribLocation(program, "vPosition");
        GLES20.glEnableVertexAttribArray(positionHandle);
        GLES20.glVertexAttribPointer(positionHandle, 2, GLES20.GL_FLOAT, false, 0, vertexBuffer);

        // 传递纹理坐标
        int textureHandle = GLES20.glGetAttribLocation(program, "vTexCoord");
        GLES20.glEnableVertexAttribArray(textureHandle);
        GLES20.glVertexAttribPointer(textureHandle, 2, GLES20.GL_FLOAT, false, 0, textureBuffer);

        // 绘制
        GLES20.glDrawArrays(GLES20.GL_TRIANGLE_STRIP, 0, 4);

        // 解绑
        GLES20.glDisableVertexAttribArray(positionHandle);
        GLES20.glDisableVertexAttribArray(textureHandle);
    }

    private int loadTexture(int resourceId) {
        final int[] textureIds = new int[1];
        GLES20.glGenTextures(1, textureIds, 0);
        int textureId = textureIds[0];

        // 绑定纹理并设置滤波参数
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureId);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);

        // 加载 Bitmap 到纹理
        Bitmap bitmap = BitmapFactory.decodeResource(context.getResources(), resourceId);
        GLUtils.texImage2D(GLES20.GL_TEXTURE_2D, 0, bitmap, 0);
        bitmap.recycle();

        return textureId;
    }

    private int createProgram() {
        // 顶点着色器代码
        String vertexShaderCode =
                "attribute vec4 vPosition;" +
                "attribute vec2 vTexCoord;" +
                "varying vec2 texCoord;" +
                "void main() {" +
                "  gl_Position = vPosition;" +
                "  texCoord = vTexCoord;" +
                "}";

        // 片段着色器代码
        String fragmentShaderCode =
                "precision mediump float;" +
                "varying vec2 texCoord;" +
                "uniform sampler2D texture;" +
                "void main() {" +
                "  gl_FragColor = texture2D(texture, texCoord);" +
                "}";

        // 编译着色器
        int vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, vertexShaderCode);
        int fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER, fragmentShaderCode);

        // 创建 OpenGL ES 程序并链接
        int program = GLES20.glCreateProgram();
        GLES20.glAttachShader(program, vertexShader);
        GLES20.glAttachShader(program, fragmentShader);
        GLES20.glLinkProgram(program);

        return program;
    }

    private int loadShader(int type, String shaderCode) {
        int shader = GLES20.glCreateShader(type);
        GLES20.glShaderSource(shader, shaderCode);
        GLES20.glCompileShader(shader);
        return shader;
    }
}
```
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
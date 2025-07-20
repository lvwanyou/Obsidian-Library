
如果你要自己实现RecyclerView加载大量图片的内存优化方案，而不依赖现代图片库(如Glide、Picasso等)，以下是完整的实现思路和关键技术点：

## 1. 图片压缩与采样

### 1.1 双阶段解码技术

这是避免OOM的核心技术，通过`inJustDecodeBounds`先获取图片尺寸信息，再计算合适的采样率：
```java

```

public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId, 
        int reqWidth, int reqHeight) {
    // 第一阶段：只获取尺寸信息
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);
    
    // 计算采样率
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
    
    // 第二阶段：实际解码
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}

public static int calculateInSampleSize(BitmapFactory.Options options,
        int reqWidth, int reqHeight) {
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;

    if (height > reqHeight || width > reqWidth) {
        final int heightRatio = Math.round((float) height / (float) reqHeight);
        final int widthRatio = Math.round((float) width / (float) reqWidth);
        inSampleSize = Math.min(heightRatio, widthRatio);
    }
    return inSampleSize;
}

135

### 1.2 按需加载

根据ImageView的实际显示尺寸加载图片，而不是加载原图：

java

// 获取ImageView的显示尺寸
imageView.getViewTreeObserver().addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
    @Override
    public boolean onPreDraw() {
        imageView.getViewTreeObserver().removeOnPreDrawListener(this);
        int width = imageView.getWidth();
        int height = imageView.getHeight();
        // 加载适合尺寸的图片
        Bitmap bitmap = decodeSampledBitmapFromResource(res, resId, width, height);
        imageView.setImageBitmap(bitmap);
        return true;
    }
});

6

## 2. 内存缓存实现

### 2.1 LruCache实现

java

// 获取应用可用内存的1/8作为缓存大小
final int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);
final int cacheSize = maxMemory / 8;

// 创建LruCache
LruCache<String, Bitmap> memoryCache = new LruCache<String, Bitmap>(cacheSize) {
    @Override
    protected int sizeOf(String key, Bitmap bitmap) {
        // 计算Bitmap占用的内存大小
        return bitmap.getByteCount() / 1024;
    }
};

// 添加图片到缓存
memoryCache.put(key, bitmap);

// 从缓存获取图片
Bitmap bitmap = memoryCache.get(key);

17

### 2.2 二级磁盘缓存

java

// 磁盘缓存实现
public class DiskCache {
    private static final long DISK_CACHE_SIZE = 1024 * 1024 * 50; // 50MB
    private DiskLruCache diskLruCache;
    
    public DiskCache(Context context) {
        File cacheDir = getDiskCacheDir(context, "bitmap");
        diskLruCache = DiskLruCache.open(cacheDir, 1, 1, DISK_CACHE_SIZE);
    }
    
    public void put(String key, Bitmap bitmap) {
        // 将Bitmap写入磁盘
        DiskLruCache.Editor editor = diskLruCache.edit(key);
        OutputStream outputStream = editor.newOutputStream(0);
        bitmap.compress(Bitmap.CompressFormat.PNG, 100, outputStream);
        editor.commit();
    }
    
    public Bitmap get(String key) {
        // 从磁盘读取Bitmap
        DiskLruCache.Snapshot snapshot = diskLruCache.get(key);
        if (snapshot != null) {
            InputStream inputStream = snapshot.getInputStream(0);
            return BitmapFactory.decodeStream(inputStream);
        }
        return null;
    }
}

3

## 3. RecyclerView专用优化

### 3.1 滑动时暂停加载

java

recyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
    @Override
    public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
        if (newState == RecyclerView.SCROLL_STATE_IDLE) {
            resumeImageLoading();
        } else {
            pauseImageLoading();
        }
    }
});

private void pauseImageLoading() {
    // 暂停所有图片加载任务
}

private void resumeImageLoading() {
    // 恢复可见项的图片加载
}

24

### 3.2 ViewHolder回收时释放资源

java

public class RecyclerImageView extends ImageView {
    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        setImageDrawable(null); // 释放图片引用
    }
}

// 在Adapter中
@Override
public void onViewRecycled(@NonNull ViewHolder holder) {
    super.onViewRecycled(holder);
    ((RecyclerImageView)holder.imageView).setImageDrawable(null);
}

24

## 4. 其他关键优化技术

### 4.1 使用RGB_565格式

java

BitmapFactory.Options options = new BitmapFactory.Options();
options.inPreferredConfig = Bitmap.Config.RGB_565; // 比ARGB_8888节省一半内存

24

### 4.2 大图单独处理

对于特别大的图片，可以考虑在新进程中加载：

xml

<activity android:name=".DetailActivity" 
          android:process=":picture_process"/>

24

### 4.3 多分辨率资源适配

提供多套分辨率资源，避免系统自动缩放：

text

res/
  drawable-mdpi/
  drawable-hdpi/
  drawable-xhdpi/
  drawable-xxhdpi/

10

## 5. 完整实现流程

1. **初始化阶段**：
    
    - 创建内存缓存LruCache
        
    - 初始化磁盘缓存
        
    - 注册RecyclerView滑动监听
        
2. **绑定数据阶段**：
    
    - 检查内存缓存 → 命中则直接使用
        
    - 未命中则检查磁盘缓存 → 命中则解码并加入内存缓存
        
    - 都未命中则启动异步任务加载原始图片
        
        - 先获取图片尺寸
            
        - 计算合适的采样率
            
        - 解码压缩后的图片
            
        - 加入内存和磁盘缓存
            
3. **回收阶段**：
    
    - ViewHolder被回收时释放图片引用
        
    - 图片不再显示时考虑从内存缓存移除
        
4. **内存管理**：
    
    - 监听onLowMemory()事件，清空缓存
        
    - 实现Bitmap复用池(BitmapPool)
        

## 性能对比指标

|优化技术|内存节省|实现复杂度|适用场景|
|---|---|---|---|
|双阶段解码|50-90%|中|所有图片加载|
|LruCache|减少重复加载|低|频繁访问的图片|
|滑动暂停|减少CPU占用|中|列表/网格视图|
|RGB_565|50%内存|低|不透明图片|
|多分辨率|避免缩放开销|高|固定尺寸资源|

自行实现完整方案需要考虑所有上述要点，并根据具体应用场景调整优化策略。现代图片库的优势在于它们已经集成了这些优化并且经过了充分测试，自行实现需要特别注意内存泄漏和并发问题。
## 先了解什么是设备密度

### 1.设备密度的计算
设备英寸是指，设备屏幕对角线英寸数。
设备密度 = 设备长(宽)分辨率 / 设备长(宽)英寸 = 每英寸的像素数
根据设备分辨率，可以计算出设备【宽高比】，然后根据 设备英寸，算出设备【宽度英寸】数。
然后设备 【宽度分辨率 / 设备宽度英寸 = 每英寸像素数】 也就是设备密度。

### 2.res目录的密度 (固定值)
默认drawable(文件夹名后不跟分辨率)----->160
ldpi -----> 120 px
mdpi -----> 160 px
hdpi -----> 240 px
xhdpi -----> 320 px
xxhdpi -----> 480 px
xxxhdpi-----> 640 px

## Bitmap到底占多大内存
从本地加载或者从网络加载可以用下面的公式计算
```
图片的长度 * 图片的宽度 * 一个像素点占用的字节数
```
如果从资源文件夹加载，会怎么样?
首先把同一张图片放进不同的资源文件夹会发生什么？
同一张图片放进不同的文件夹，图片会被压缩,压缩比例的计算公式：
```
scale = (float) targetDensity / density;
```
targetDensity：设备屏幕像素密度 dpi
density：图片对应的文件夹的像素密度 dpi

其中density和Bitmap存放的资源目录有关，不同的资源目录有不同的值

| density | 0.75 | 1 | 1.5 | 2 |
| -| -| -| -|-|
| densityDpi | 120 | 160 | 240 | 320 |
| DpiFolder| ldpi| mdpi| hdpi|xhdpi|


所以Bitmap在资源目录中的计算方式为
```
Bitmap内存占用 ≈ 像素数据总大小 = 图片宽 × 图片高× (当前设备密度dpi/图片所在文件夹对应的密度dpi）^2 × 每个像素的字节大小
```

## 如何优化
编码
压缩（质量压缩，采样压缩，宽高缩放压缩）
复用
下面我们一个个的来讲这些优化

### 编码
![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/8/19/16ca99cc6a78c80a~tplv-t2oaga2asx-no-mark:1280:960:0:0.awebp)

其中，A代表透明度；R代表红色；G代表绿色；B代表蓝色。

ALPHA_8
表示8位Alpha位图,即A=8,一个像素点占用1个字节,它没有颜色,只有透明度
ARGB_4444
表示16位ARGB位图，即A=4,R=4,G=4,B=4,一个像素点占4+4+4+4=16位，2个字节
ARGB_8888
表示32位ARGB位图，即A=8,R=8,G=8,B=8,一个像素点占8+8+8+8=32位，4个字节
RGB_565
表示16位RGB位图,即R=5,G=6,B=5,它没有透明度,一个像素点占5+6+5=16位，2个字节

也即是说我们可以通过改变图片格式，来改变每个像素占用字节数，来改变占用的内存，看下面代码
```
BitmapFactory.Options options = new BitmapFactory.Options();
        //不获取图片，不加载到内存中，只返回图片属性
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeFile(photoPath, options);
        //图片的宽高
        int outHeight = options.outHeight;
        int outWidth = options.outWidth;
        Log.d("mmm", "图片宽=" + outWidth + "图片高=" + outHeight);
        //图片格式压缩
        options.inPreferredConfig = Bitmap.Config.RGB_565;
        options.inJustDecodeBounds = false;
        Bitmap bitmap = BitmapFactory.decodeFile(photoPath, options);
        float bitmapsize = getBitmapsize(bitmap);
        Log.d("mmm","压缩后：图片占内存大小" + bitmapsize + "MB / 宽度=" + bitmap.getWidth() + "高度=" + bitmap.getHeight());

```

log:
```
07-09 11:10:46.042 15312-15312/D/mmm: 原图：图片占内存大小=45.776367MB / 宽度=4000高度=3000
07-09 11:10:46.043 15312-15312/D/mmm: 图片宽=4000图片高=3000
07-09 11:10:46.367 15312-15312/D/mmm: 压缩后：图片占内存大小22.887695MB / 宽度=4000高度=3000
```

宽高没变，我们改变了图片的格式，从ARGB_8888 变成了RGB_565 ，像素占用字节数减少了一般，根据log 内存也减少了一半，这种方式可行
注意：由于ARGB_4444的画质惨不忍睹，一般假如对图片没有透明度要求的话，可以改成RGB_565，相比ARGB_8888将节省一半的内存开销。

### 图片的压缩
一、更换图片格式：
Android目前常用的图片格式有png，jpeg和webp，
png：无损压缩图片格式，支持Alpha通道，Android切图素材多采用此格式

 jpeg：有损压缩图片格式，不支持背景透明，适用于照片等色彩丰富的大图压缩，不适合logo

 webp：是一种同时提供了有损压缩和无损压缩的图片格式，派生自视频编码格式VP8，从
 谷歌官网
 来看，无损webp平均比png小26%，有损的webp平均比jpeg小25%~34%，无损webp支持Alpha通道，有损webp在一定的条件下同样支持，有损webp在Android4.0（API 14）之后支持，无损和透明在Android4.3（API18）之后支持

二、质量压缩：
通过 quality 来压缩
质量压缩并不会改变图片在内存中的大小，仅仅会减小图片所占用的磁盘空间的大小，因为质量压缩不会改变图片的分辨率，而图片在内存中的大小是根据
 width*height*一个像素的所占用的字节数计算的，宽高没变，在内存中占用的大小自然不会变，质量压缩的原理是通过改变图片的位深和透明度来减小图片占用的磁盘空间大小，所以不适合作为缩略图，可以用于想保持图片质量的同时减小图片所占用的磁盘空间大小，png是无损压缩，所以设置quality无效

  ```

/**
 * 质量压缩
 *
 * @param format  图片格式 jpeg,png,webp
 * @param quality 图片的质量,0-100,数值越小质量越差
 */
public static void compress(Bitmap.CompressFormat format, int quality) {
    File sdFile = Environment.getExternalStorageDirectory();
    File originFile = new File(sdFile, "originImg.jpg");
    Bitmap originBitmap = BitmapFactory.decodeFile(originFile.getAbsolutePath());
    ByteArrayOutputStream bos = new ByteArrayOutputStream();
    originBitmap.compress(format, quality, bos);
    try {
        FileOutputStream fos = new FileOutputStream(new File(sdFile, "resultImg.jpg"));
        fos.write(bos.toByteArray());
        fos.flush();
        fos.close();
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }

 ```

 三、采样率压缩：
采样率压缩是通过设置BitmapFactory.Options.inSampleSize，来减小图片的分辨率，进而减小图片所占用的磁盘空间和内存大小。设置的inSampleSize会导致压缩的图片的宽高都为1/inSampleSize

```
/**
 * 
 * @param inSampleSize  可以根据需求计算出合理的inSampleSize
 */
public static void compress(int inSampleSize) {
    File sdFile = Environment.getExternalStorageDirectory();
    File originFile = new File(sdFile, "originImg.jpg");
    BitmapFactory.Options options = new BitmapFactory.Options();
    //设置此参数是仅仅读取图片的宽高到options中，不会将整张图片读到内存中，防止oom
    options.inJustDecodeBounds = true;
    Bitmap emptyBitmap = BitmapFactory.decodeFile(originFile.getAbsolutePath(), options);
 
    options.inJustDecodeBounds = false;
    options.inSampleSize = inSampleSize;
    Bitmap resultBitmap = BitmapFactory.decodeFile(originFile.getAbsolutePath(), options);
    ByteArrayOutputStream bos = new ByteArrayOutputStream();
    resultBitmap.compress(Bitmap.CompressFormat.JPEG, 100, bos);
    try {
        FileOutputStream fos = new FileOutputStream(new File(sdFile, "resultImg.jpg"));
        fos.write(bos.toByteArray());
        fos.flush();
        fos.close();
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
```

四、缩放压缩：
通过减少图片的像素来降低图片的磁盘空间大小和内存大小，可以用于缓存缩略图。
```

public void compress(View v) {
    File sdFile = Environment.getExternalStorageDirectory();
    File originFile = new File(sdFile, "originImg.jpg");
    Bitmap bitmap = BitmapFactory.decodeFile(originFile.getAbsolutePath());
    //设置缩放比
    int radio = 8;
    Bitmap result = Bitmap.createBitmap(bitmap.getWidth() / radio, bitmap.getHeight() / radio, Bitmap.Config.ARGB_8888);
    Canvas canvas = new Canvas(result);
    RectF rectF = new RectF(0, 0, bitmap.getWidth() / radio, bitmap.getHeight() / radio);
    //将原图画在缩放之后的矩形上
    canvas.drawBitmap(bitmap, null, rectF, null);
    ByteArrayOutputStream bos = new ByteArrayOutputStream();
    result.compress(Bitmap.CompressFormat.JPEG, 100, bos);
    try {
        FileOutputStream fos = new FileOutputStream(new File(sdFile, "sizeCompress.jpg"));
        fos.write(bos.toByteArray());
        fos.flush();
        fos.close();
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

五、总结：
1、使用webp格式的图片可以在保持清晰度的情况下减小图片的磁盘大小，是一种比较优秀的，google推荐的图片格式

 2、质量压缩可以减小图片占用的磁盘空间，不会减小在内存中的大小

 3、采样率压缩可以通过改变分辨率来减小图片所占用的磁盘空间和内存空间大小，但是采样率只能设置2的n次方，可能图片的最优比例在中间

 4、尺寸压缩同样也是通过改变分辨率来减小图片所占用的磁盘空间和内存空间大小，缩放的尺寸没有什么限制

 5、jni调用jpeg库来弥补安卓系统skia框架的不足，也是比较优秀的解决方式（方法会在后期补充）

 既然尺寸压缩和采样率压缩都是通过改变图片的分辨率来降低大小，有什么区别吗？

 其中一个原因已经说明了，采样率的压缩比例会受到限制，尺寸压缩不会，但是采样率压缩的清晰度会比尺寸压缩的清晰度要好一些


### 复用
图片复用指的是inBitmap这个属性
这个属性又什么作用？
不使用这个属性，你加载三张图片，系统会给你分配三份内存空间，用于分别储存这三张图片
如果用了inBitmap这个属性，加载三张图片，这三张图片会指向同一块内存，而不用开辟三块内存空间
inBitmap的限制

3.0-4.3

复用的图片大小必须相同
编码必须相同


4.4以上

复用的空间大于等于即可
编码不必相同


不支持WebP
图片复用，这个属性必须设置为true；
options.inMutable = true;

### 图片到底储存在哪里

| 2.3- | 3.0-4.4 | 5.0-7.1 | 8.0 |
| -| -| -| -|
| Bitmap对象 | java Heap | java Heap | java Heap |
| 像素数据| Native Heap| java Heap| Native Heap|
| 迁移原因| -| 解决Native Bitmap内存泄露| 共享整个系统的内存减少OOM|


### 如何不压缩加载高清图
如果有需求，要求我们既不能压缩图片，又不能发生oom怎么办，这种情况我们需要加载图片的一部分区域来显示，下面我们来了解一下BitmapRegionDecoder这个类，加载图片的一部分区域，他的用法很简单
```
//支持传入图片的路径，流和图片修饰符等
   BitmapRegionDecoder mDecoder = BitmapRegionDecoder.newInstance(path, false);
//需要显示的区域就有由rect控制，options来控制图片的属性
    Bitmap bitmap = mDecoder.decodeRegion(mRect, options);
```
由于要显示一部分区域，所以要有手势的控制，方便上下的滑动，需要自定义控件，而自定义控件的思路也很简单 1 提供图片的入口 2 重写onTouchEvent， 根据手势的移动更新显示区域的参数 3 更新区域参数后，刷新控件重新绘制
自定义view代码
```
public class BigImageView extends View {

    private BitmapRegionDecoder mDecoder;
    private int mImageWidth;
    private int mImageHeight;
    //图片绘制的区域
    private Rect mRect = new Rect();
    private static final BitmapFactory.Options options = new BitmapFactory.Options();

    static {
        options.inPreferredConfig = Bitmap.Config.RGB_565;
    }

    public BigImageView(Context context) {
        super(context);
        init();
    }

    public BigImageView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public BigImageView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {

    }

    /**
     * 自定义view的入口，设置图片流
     *
     * @param path 图片路径
     */
    public void setFilePath(String path) {
        try {
            //初始化BitmapRegionDecoder
            mDecoder = BitmapRegionDecoder.newInstance(path, false);
            BitmapFactory.Options options = new BitmapFactory.Options();
            //便是只加载图片属性，不加载bitmap进入内存
            options.inJustDecodeBounds = true;
            BitmapFactory.decodeFile(path, options);
            //图片的宽高
            mImageWidth = options.outWidth;
            mImageHeight = options.outHeight;
            Log.d("mmm", "图片宽=" + mImageWidth + "图片高=" + mImageHeight);

            requestLayout();
            invalidate();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        //获取本view的宽高
        int measuredHeight = getMeasuredHeight();
        int measuredWidth = getMeasuredWidth();


        //默认显示图片左上方
        mRect.left = 0;
        mRect.top = 0;
        mRect.right = mRect.left + measuredWidth;
        mRect.bottom = mRect.top + measuredHeight;
    }

    //第一次按下的位置
    private float mDownX;
    private float mDownY;

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                mDownX = event.getX();
                mDownY = event.getY();
                break;
            case MotionEvent.ACTION_MOVE:
                float moveX = event.getX();
                float moveY = event.getY();
                //移动的距离
                int xDistance = (int) (moveX - mDownX);
                int yDistance = (int) (moveY - mDownY);
                Log.d("mmm", "mDownX=" + mDownX + "mDownY=" + mDownY);
                Log.d("mmm", "movex=" + moveX + "movey=" + moveY);
                Log.d("mmm", "xDistance=" + xDistance + "yDistance=" + yDistance);
                Log.d("mmm", "mImageWidth=" + mImageWidth + "mImageHeight=" + mImageHeight);
                Log.d("mmm", "getWidth=" + getWidth() + "getHeight=" + getHeight());
                if (mImageWidth > getWidth()) {
                    mRect.offset(-xDistance, 0);
                    checkWidth();
                    //刷新页面
                    invalidate();
                    Log.d("mmm", "刷新宽度");
                }
                if (mImageHeight > getHeight()) {
                    mRect.offset(0, -yDistance);
                    checkHeight();
                    invalidate();
                    Log.d("mmm", "刷新高度");
                }
                break;
            case MotionEvent.ACTION_UP:
                break;
            default:
        }
        return true;
    }


    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Bitmap bitmap = mDecoder.decodeRegion(mRect, options);
        canvas.drawBitmap(bitmap, 0, 0, null);
    }

    /**
     * 确保图不划出屏幕
     */
    private void checkWidth() {


        Rect rect = mRect;
        int imageWidth = mImageWidth;
        int imageHeight = mImageHeight;

        if (rect.right > imageWidth) {
            rect.right = imageWidth;
            rect.left = imageWidth - getWidth();
        }

        if (rect.left < 0) {
            rect.left = 0;
            rect.right = getWidth();
        }
    }

    /**
     * 确保图不划出屏幕
     */
    private void checkHeight() {

        Rect rect = mRect;
        int imageWidth = mImageWidth;
        int imageHeight = mImageHeight;

        if (rect.bottom > imageHeight) {
            rect.bottom = imageHeight;
            rect.top = imageHeight - getHeight();
        }

        if (rect.top < 0) {
            rect.top = 0;
            rect.bottom = getHeight();
        }
    }
}


```

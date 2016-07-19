---
layout: post
title: "Android位图总结"
date: 2010-12-18 21:39:34 +0800
comments: true
categories: android
---

由于项目中牵扯到了对位图(android.graphics.Bitmap)的操作，于是对照Android的参考文档详细地研究了一下Android提供的位图相关功能。
## 一、对位图的获取
在Android的SDK中提供了一个BitmapFactory 类。采用此类的几个方法能够从一个文件路径或者输入流中得到位图。

- 包：android.graphics
- 类：BitmapFactory
- Android SDK中的简介：Creates Bitmap objects from various sources, including files, streams, and byte-arrays.

此类其提供了以下方法来获取Bitmap，这些方法都是静态的:

1. decodeByteArray(byte[] data, int offset, int length)从一个字节数组中得到数据转换为位图。
2. decodeFile(String pathName)由一个图像文件路径的得到位图（支持jpg、png、bmp格式）
3. decodeFileDescriptor(FileDescriptor fd)由一个文件描述符得到位图。
4. decodeResource(Resources res, int id)由资源ID获取位图
5. decodeStream(InputStream is)由图片输入流获取位图。

这里，由于Android系统本身对apk程序的限制，当加载大图片时会产生OutOfMemoryError。可以采取两种办法解决：
	
- 先使用android.provider.MediaStore.Images.Media类的query(ContentResolver cr, Uri uri, String[] projection, String where, String orderBy)方法获得对应的图片所对应的id，然后使用android.provider.MediaStore.Images.Thumbnails类的queryMiniThumbnail(ContentResolver cr, long origId, int kind, String[] projection)获得图片对应的缩略图的路径，再使用BitmapFactory的decodeFile(String pathname)获取到图片的缩略图，这样即可避免内存溢出的发生。
- 采取压缩图片的方法对图片进行压缩。这里主要还是使用BitmapFactory类，对于上面提到过的能够获取位图的几个方法，都对应着另一个多了一个参数的方法，例如decodeFile(String pathName)对应着decodeFile(String pathName, BitmapFactory.Options opts)这个方法，其他的可查阅参考文档。其中，通过引入opts这个参数，对其进行相关的设置，最后能得到一个被压缩的图片。

## 二、对位图信息的获取
要获取位图信息，比如位图大小、是否包含透明度、颜色格式等，获取得到Bitmap就迎刃而解了，这些信息在Bitmap的函数中可以轻松获取到。

Android SDK中对Bitmap有详细说明，这里辅助说明以下2点：

1. 在Bitmap中对RGB颜色格式使用Bitmap.Config定义，仅包括ALPHA_8、ARGB_4444、ARGB_8888、RGB_565，缺少了一些其他的，比如说RGB_555，在开发中可能需要注意这个小问题；
2. Bitmap还提供了compress()接口来压缩图片，不过Android SDK只支持PNG、JPG格式的压缩；其他格式的需要Android开发人员自己补充了。

## 三、位图的显示
位图的显示，一般可以在界面中放置ImageView控件，然后调用android.widget.ImageView类的setImageBitmap(Bitmap bm)方法将位图显示出来。另外，可以使用核心类android.graphics.Canvas的drawBirmap()方法显示位图，或者借助于BitmapDrawable来将Bitmap绘制到Canvas。

## 四、位图的缩放
位图的缩放，在Android SDK中提供了2种方法：

1. 将一个位图按照需求重画一遍，画后的位图就是我们需要的了，与位图的显示几乎一样：drawBitmap(Bitmap bitmap, Rect src, Rect dst, Paint paint)
2. 在原有位图的基础上，缩放原位图，创建一个新的位图： createBitmap(Bitmap source, int x, int y, int width, int height, Matrix m, boolean filter)
归结到底，位图缩放的本质就是将原始位图按照需求显示出来，创造一张新的位图。

## 五、位图的旋转
位图的旋转，离不开Matrix。Matrix在线性代数中都学习过，Android SDK提供了Matrix类，可以通过各种接口来设置矩阵。例子如下“

<pre>
Matrix matrix = new Matrix();matrix.setRotate(90,120,130);canvas.drawBitmap(mbmpTest, matrix, mPaint);
</pre>

除了这种方法之外，我们也可以在使用Bitmap提供的函数如下：

<pre>
public static Bitmap createBitmap (Bitmap source, int x, int y, int width, int height, Matrix m, boolean filter)
</pre>

在原有位图旋转的基础上，创建新位图。

## 六、位图的截取
对于位图的规则形状如矩形的截取，很容易通过Matrix进行实现。例子如下：

<pre>
Bitmap croppedImage;
// 取得裁剪的矩形
Rect r = mCrop.getCropRect(); 
if(r == null) return;
int width = r.width();
int height = r.height();
// 生成裁剪的图片
croppedImage = Bitmap.createBitmap(width, height, Bitmap.Config.RGB_565);
Canvas canvas = new Canvas(croppedImage);
Rect dstRect = new Rect(0, 0, width, height);
canvas.drawBitmap(mBitmap, r, dstRect, null);
</pre>

而对于位图的不规则形状的截取，找不到很好的解决办法。曾想过直接对像素进行操作，最终因为算法的的复杂而放弃。一次偶然的机会，在阅读一篇博文时，我发现了使用android.graphics.Canvas的clipPath方法不仅仅可以达到高亮显示路径的目的，也能够达到任意形状截取图片的目的，从而解决了此问题。

以上就是对Android的位图功能进行的总结，以及一些自己实践中得出的想法和结论。更深入的内容需要仔细地去阅读研究Android SDK文档并付之于实践。
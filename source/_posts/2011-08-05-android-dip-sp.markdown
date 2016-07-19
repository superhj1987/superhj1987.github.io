---
layout: post
title: "Android中dip、dp、sp、pt和px的区别"
date: 2011-08-05 21:39:34 +0800
comments: true
categories: android
---

Android有以下几个度量单位：

   - dip: device independent pixels(设备独立像素). 不同设备有不同的显示效果,这个和设备硬件有关，一般我们为了支持WVGA、HVGA和QVGA 推荐使用这个，不依赖像素。 
   
   - px: pixels(像素). 不同设备显示效果相同，一般我们HVGA代表320x480像素，这个用的比较多。 

   - pt: point，是一个标准的长度单位，1pt＝1/72英寸，用于印刷业，非常简单易用； 

   - sp: scaled pixels(放大像素). 主要用于字体显示best for textsize。 

由此，根据 google 的建议，TextView 的字号最好使用 sp 做单位，而且查看TextView的源码可知Android默认使用sp作为字号单位。

关于换算（以 sp 和 pt 为例） 
查看　TextView 等类的源码，可知： 
<pre>
case COMPLEX_UNIT_PX: 
      return value; 
case COMPLEX_UNIT_SP: 
      return value * metrics.scaledDensity; 
case COMPLEX_UNIT_PT: 
      return value * metrics.xdpi * (1.0f/72); 

－－－－－－－－－－－－－－－－－－－－－－－－－－ 
scaledDensity = DENSITY_DEVICE / (float) DENSITY_DEFAULT; 
xdpi = DENSITY_DEVICE; 

－－－－－－－－－－－－－－－－－－－－－－－－－－ 
DENSITY_DEFAULT = DENSITY_MEDIUM = 160; 

============================================ 
</pre>
   所以： 假设　pt 和 sp 取相同的值 1，则可设 1pt 和 1sp 之间系数为 x， 
<pre>
1 * DENSITY_DEVICE / 72 = x * 1 * DENSITY_DEVICE / 160  =&gt; 
x = 160 / 72 = 2.2222 
</pre>
也就是说在 Android 中，  1pt 大概等于 2.22sp 

以上供参考，如果 UI 能够以 sp 为单位提供设计是最好的，如果设计中没有 sp 
的概念，则开发人员也可以通过适当的换算取近似值。 

   过去，程序员通常以像素为单位设计计算机用户界面。例如，定义一个宽度为300像素的表单字段，列之间的间距为5个像素，图标大小为16×16像素 等。这样处理的问题在于，如果在一个每英寸点数（dpi）更高的新显示器上运行该程序，则用户界面会显得很小。在有些情况下，用户界面可能会小到难以看清 内容。 

与分辨率无关的度量单位可以解决这一问题。Android支持下列所有单位。 

- px（像素）：屏幕上的点。 

- in（英寸）：长度单位。 

- mm（毫米）：长度单位。 

- pt（磅）：1/72英寸。 

- dp（与密度无关的像素）：一种基于屏幕密度的抽象单位。在每英寸160点的显示器上，1dp = 1px。 

- dip：与dp相同，多用于android/ophone示例中。 

- sp（与刻度无关的像素）：与dp类似，但是可以根据用户的字体大小首选项进行缩放。 

为了使用户界面能够在现在和将来的显示器类型上正常显示，建议大家始终使用sp作为文字大小的单位，将dip作为其他元素的单位。当然，也可以考虑使用矢量图形，而不是用位图。
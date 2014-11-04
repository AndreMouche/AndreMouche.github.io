---
layout: post
title: "使用GraphicsMagick图片旋转"
keywords: ["GraphicsMagick", "RotateImage"]
description: "rotate Image with graphicsMagick "
category: "GraphicsMagick"
tags: ["GraphicsMagick"]

---

##旋转图片
###命令行
```
 gm convert data/travel.jpg -rotate 120 rotate.jpg
```
##[接口](http://www.graphicsmagick.org/api/shear.html#rotateimage)
```
Image *RotateImage( const Image *image, const double degrees,
                    ExceptionInfo *exception );
```

* 该方法通过拷贝已存在的图片对象创建一个旋转图片对象。
* degrees为正则顺时针转，为负逆时针转。
* 旋转后的图片一般大于原图片，且包含一些空白三角。空白三角的颜色由原图的background_color属性决定。
* 方法在内存中为新图创建新的图片对象，且返回该对象的指针。

##Sample

### Code:
[rotate.cpp](https://github.com/AndreMouche/GraphicsMagicUsage/blob/master/rotate.cpp)
###效果

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsMagicUsage/master/data/rotate.jpg" alt="rotate.jpg" title="rotate.jpg" width="400" />

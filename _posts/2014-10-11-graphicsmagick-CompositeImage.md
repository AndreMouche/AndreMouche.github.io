---
layout: post
title: "使用GraphicsMagick拼接图片"
keywords: ["GraphicsMagick", "CompositeImage"]
description: "Composite Image with graphicsMagick "
category: "GraphicsMagick"
tags: ["GraphicsMagick"]

---

##使用GraphicsMagick拼接图片(打图片水印)
通过使用[CompositeImage](http://www.graphicsmagick.org/api/composite.html)给图片打图片水印

##CompositeImage
###语法

```
MagickPassFail CompositeImage( Image *canvas_image, const CompositeOperator compose,
                               const Image *composite_image, const long x_offset,
                               const long y_offset );

```
该接口将第二张图片（composite_image）拼接到第一张图片(canvas_image)的指定偏移上（x_offset,y_offset）

###参数
**canvas_image**

画布，顾名思义，需要被修改的图对象

**compose**

该参数指定了[拼接算法](http://www.graphicsmagick.org/api/types.html#compositeoperator)，主要有以下几种方式：

算法 | 描述
----  | ------
OverCompositeOp |   结果为composite_image覆盖在canvas_image指定位置上
InCompositeOp | 看不出与OverCompositeOp的区别 
OutCompositeOp | 测试结果发现得到的图片为canvas_image
AtopCompositeOp | 看不出与OverCompositeOp的区别 
XorCompositeOp | 结果为composite_image覆盖在canvas_image指定位置上，其中重叠部分为黑色
PlusCompositeOp| 结果为composite_image覆盖在canvas_image指定位置上，其中重叠部分为图片数据之和，超过255以255记
MinusCompositeOp|结果为composite_image覆盖在canvas_image指定位置上，其中重叠部分为图片数据canvas_image-composite_image，小于0的记0
AddCompositeOp|结果为composite_image覆盖在canvas_image指定位置上，其中重叠部分为图片数据之和，超过255对256取模
SubtractCompositeOp|结果为composite_image覆盖在canvas_image指定位置上，其中重叠部分为图片数据canvas_image-composite_image，小于0的对256取模
DifferenceCompositeOp| 用于比较两图是否相同（测试结果貌似不太符合预计?）
BumpmapCompositeOp|在canvas_image指定位置上打composite_image阴影
CopyCompositeOp|结果为composite_image覆盖在canvas_image指定位置上，哑光信息被忽略。
CopyRedCompositeOp|结果为提取composite_image的red layer覆盖在canvas_image指定位置上
CopyGreenCompositeOp|结果为提取composite_image的green layer覆盖在canvas_image指定位置上
CopyBlueCompositeOp|结果为提取composite_image的blue layer覆盖在canvas_image指定位置上
CopyOpacityCompositeOp|结果为提取composite_image的matte layer覆盖在canvas_image指定位置上
ClearCompositeOp|原图？
DissolveCompositeOp|原图？
DisplaceCompositeOp|对应位置有composite_image透明的阴影？
ModulateCompositeOp|调节[ HSL space](http://zh.wikipedia.org/wiki/HSL%E5%92%8CHSV%E8%89%B2%E5%BD%A9%E7%A9%BA%E9%97%B4)的亮度？
ThresholdCompositeOp|打阴影？
NoCompositeOp|不做任何事
DarkenCompositeOp| 结果为composite_image比canvas_image暗部分覆盖在canvas_image指定位置上？
LightenCompositeOp|结果为composite_image比canvas_image亮部分覆盖在canvas_image指定位置上？
...|...

**composite_image**

复合图对象

**x_offset**

原图上的拼接列偏移

**y_offset**

原图上的拼接行偏移

##Sample

参考GraphicsMagick [CompositeImage](http://www.graphicsmagick.org/api/composite.html)给图片打图片水印

加透明度后的水印

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsMagicUsage/master/data/disslove.jpg" alt="disslove.jpg" title="disslove.jpg" width="400" />

###Code 

 [composite.cpp](https://github.com/AndreMouche/GraphicsMagicUsage/blob/master/composite.cpp)
 

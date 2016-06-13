---
layout: post
title: "使用GraphicsMagick图片渐进显示设置"
keywords: ["GraphicsMagick", "WriteImage"]
description: "Convert Image with graphicsMagick "
category: "GraphicsMagick"
tags: ["GraphicsMagick"]
date: 2014-10-20
comments: true
---

## 设置图片渐进显示

当网络差时，图片载入方式由模糊到清晰。该功能由WriteImage (imageInfo,image)中imageInfo的interlace参数指定：

```
typedef enum
 {
   UndefinedInterlace,
   NoInterlace,        //无渐进
   LineInterlace,      //有渐进
   PlaneInterlace,     //不可用
   PartitionInterlace  //不可用
 } InterlaceType;

```

## 检查图片是否开启渐进显示：

```
identify -verbose test.jpg | grep Interlace
```

## 渐进显示案例
[convert.cpp](https://github.com/AndreMouche/GraphicsStudy/blob/master/GraphicsMagicUsage/convert.cpp)

### LineInterlace

#### 原图

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/lineinterlace.jpg" alt="lineinterlace.jpg" title="lineinterlace.jpg" width="600" />


#### 显示效果

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/interlace_1.PNG" alt="interlace_1.PNG" title="interlace_1.PNG" width="600" />

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/interlace2.PNG" alt="interlace2.PNG" title="interlace2.PNG" width="600" />

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/interlace3.PNG" alt="interlace3.PNG" title="interlace3.PNG" width="600" />

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/interlace5.PNG" alt="interlace5.PNG" title="interlace5.PNG" width="600" />

### NoneInterlace

#### 原图

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/nointerlace.jpg" alt="nointerlace.jpg" title="nointerlace.jpg" width="600" />

#### 渐进效果
<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/nointerlace1.PNG" alt="nointerlace1.PNG" title="nointerlace1.PNG" width="600" />

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/nointerlace2.PNG" alt="nointerlace2.PNG" title="nointerlace2.PNG" width="600" />

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/nointerlace3.PNG" alt="nointerlace3.PNG" title="nointerlace3.PNG" width="600" />

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/nointerlace4.PNG" alt="nointerlace4.PNG" title="nointerlace4.PNG" width="600" />

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/nointerlace5.PNG" alt="nointerlace5.PNG" title="nointerlace5.PNG" width="600" />

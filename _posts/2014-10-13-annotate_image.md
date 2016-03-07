---
layout: post
title: "使用GraphicsMagick打文字水印"
keywords: ["GraphicsMagick", "AnnotateImage"]
description: "annotate pic  with graphicsMagick "
category: "GraphicsMagick"
tags: ["GraphicsMagick"]
comments: true
---

## 使用GraphicsMagick打文字水印

使用[AnnotateImage](http://www.graphicsmagick.org/api/annotate.html)给图片打文字水印

## 语法

```
unsigned int AnnotateImage( Image *image, DrawInfo *draw_info );
```

### DrawInfo

[DrawInfo](http://www.graphicsmagick.org/api/types.html#drawinfo)数据结构用来支持通过使用绘图命令给图片注释
主要方法

方法|说明
----- |-------
void 	GetDrawInfo (const ImageInfo *, DrawInfo *)|使用默认参数分配一个DrawInfo对象

DrawInfo *CloneDrawInfo( const ImageInfo *image_info, const DrawInfo *draw_info )|分配一个对象，并从其它对象拷贝所有值，若参数为空，则使用默认参数初始化对象。

void DestroyDrawInfo( DrawInfo *draw_info )|[DestroyDrawInfo](http://www.graphicsmagick.org/api/render.html#destroydrawinfo)释放DrawInfo空间

DrawImage( Image *image, const DrawInfo *draw_info )|在当前图上画东西，这个东西可以是一个字符串，也可以是文件名。用@作为前缀表示是个文件名，对因文件内容将被画在图片上。注意：该接口已经很老了，可以使用Draw这个方法替代。

MD，连个Sample的搜不到，半路出家玩图片的哪懂那些专业术语，想杀人XXXXXXXXXXXX

## 参数说明：

参数 |类型 |说明
----- |-------- |--------
font|char *|渲染文字使用的字体所在文件路径，不可为空
gravity|(NorthWest,North,NorthEast, West,Center,East, SouthWest,South,SouthEast)|渲染文字所在位置重心，左上，中上，右上，左中，中间，右中，左下，中下，右下
pointsize|double|渲染文字大小
geometry|char *|文字编码后所占矩形的大小，sample "+100+100"

## 案例
代码[annotate.cpp](https://github.com/AndreMouche/GraphicsStudy/blob/master/GraphicsMagicUsage/annotate.cpp)
效果：

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsStudy/master/GraphicsMagicUsage/data/annotate.jpg" alt="annotate.jpg" title="annotate.jpg" width="400" />





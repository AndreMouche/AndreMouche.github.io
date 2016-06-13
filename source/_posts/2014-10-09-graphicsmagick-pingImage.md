---
layout: post
title: "使用GraphicsMagick提取图片基本信息"
keywords: ["GraphicsMagick", "PingImage"]
description: "Get pic info with graphicsMagick "
category: "GraphicsMagick"
tags: ["GraphicsMagick"]
date: 2014-10-09
comments: true
---

## 使用GraphicsMagick提取图片基本信息

通过使用[PingImage](http://www.graphicsmagick.org/api/constitute.html#pingimage)获取图片基本属性。

## PingImage

### 简介

```
Image *PingImage( const ImageInfo *image_info, ExceptionInfo *exception );
```

### 说明

* 它返回指定图片除像素（Pixels）以外的所有属性。
* 相比ReadImage它更快，且使用更少内存。
* 执行失败时，返回Image为NULL，且通过exception返回详细失败信息。

### 参数

**image_info **
 该参数为由文件或文件名初始化的ImageInfo对象



## Sample

参考GraphicsMagick[官方文档](http://www.graphicsmagick.org/api/types.html#image)
提取图片基本信息

### Code 

image_info.cpp

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <sys/types.h>
#include <magick/api.h>
#include<iostream>
using namespace std;

int main ( int argc, char **argv )
{
    Image
        *image = (Image *) NULL;

    char
        infile[MaxTextExtent],
        outfile[MaxTextExtent];

    int
        arg = 1,
            exit_status = 0;

    ImageInfo
        *imageInfo;

    ExceptionInfo
        exception;

    InitializeMagick(NULL);
    imageInfo=CloneImageInfo(0);
    GetExceptionInfo(&exception);

    if (argc != 2)
    {
        (void) fprintf ( stderr, "Usage: %s infile\n", argv[0] );
        (void) fflush(stderr);
        exit_status = 1;
        goto program_exit;
    }

    (void) strncpy(infile, argv[arg], MaxTextExtent-1 );

    (void) strcpy(imageInfo->filename, infile);
    
    if((image = PingImage(imageInfo,&exception)) == NULL) {
          printf("PingImage fail, path = %s",infile);
          exit_status = -1;
          goto program_exit;
    }
    
    cout << "image Type:" << image->magick << endl;
    cout << "image width:" << image->columns << endl;
    cout << "image height:" << image->rows << endl;
    cout << "Image colorspace:" << image->colorspace << endl;

    /*switch(image->colorspace){
       
    
    }*/




program_exit:

    if (image != (Image *) NULL)
        DestroyImage(image);

    if (imageInfo != (ImageInfo *) NULL)
        DestroyImageInfo(imageInfo);
    DestroyMagick();

    return exit_status;
}

```

### 编译

```
g++ image_info.cpp `GraphicsMagick-config --cppflags --ldflags --libs`
```

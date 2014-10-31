---
layout: post
title: "使用GraphicsMagick添加webp支持"
keywords: ["GraphicsMagick", "webp"]
description: "support webp  in graphicsMagick "
category: "GraphicsMagick"
tags: ["GraphicsMagick"]

---

##WEBP支持
###安装配置

####software

   * GraphicsMagick 1.3.20 
   * https://webp.googlecode.com/files/libwebp-0.4.0.tar.gz

##install

   * 编译安装libwebp
   * 编译安装GraphicsMagicks时，指定libwebp安装位置：
    ```
    ./configure  --prefix=/home/wuxuelian/software/graphicsmagick/build CPPFLAGS='-I/home/wuxuelian/software/libwebp/build/include' LDFLAGS='-L/home/wuxuelian/software/libwebp/build/lib' 

    ```
    
确保Configure结果中WEBP那项为YES

```
GraphicsMagick is configured as follows. Please verify that this
configuration matches your expectations.

Host system type : x86_64-apple-darwin14.0.0
Build system type : x86_64-apple-darwin14.0.0

Option            Configure option           	Configured value
-----------------------------------------------------------------
Shared libraries  --enable-shared=no    	no
Static libraries  --enable-static=yes    	yes
GNU ld            --with-gnu-ld=no        	no
Quantum depth     --with-quantum-depth=8 	8
Modules           --with-modules=no        	no

Delegate Configuration:
BZLIB             --with-bzlib=yes          	yes
DPS               --with-dps=yes              	no
FlashPIX          --with-fpx=no              	no
FreeType 2.0      --with-ttf=yes          	yes
Ghostscript       None                   	gs (unknown)
Ghostscript fonts --with-gs-font-dir=default    none
Ghostscript lib   --with-gslib=no       	no
JBIG              --with-jbig=yes        	no
WEBP              --with-webp=yes        	yes
JPEG v1           --with-jpeg=yes        	yes
JPEG-2000         --with-jp2=yes          	no
LCMS v1           --with-lcms=yes        	no
LCMS v2           --with-lcms2=yes        	no
LZMA              --with-lzma=yes        	no (failed tests)
Magick++          --with-magick-plus-plus=yes 	yes
PERL              --with-perl=no            	no
PNG               --with-png=yes          	yes (-lpng16)
TIFF              --with-tiff=yes        	no
TRIO              --with-trio=yes        	no
Windows fonts     --with-windows-font-dir=	none
WMF               --with-wmf=yes          	no
X11               --with-x=             	no
XML               --with-xml=yes          	yes
ZLIB              --with-zlib=yes        	yes

X11 Configuration:

  Not using X11.

Options used to compile and link:
  CC       = gcc
  CFLAGS   = -g -O2 -Wall -D_THREAD_SAFE
  CPPFLAGS = -I/usr/local/Cellar/freetype/2.5.3_1/include/freetype2 -I/usr/local/include/libxml2
  CXX      = g++
  CXXFLAGS = -D_THREAD_SAFE
  DEFS     = -DHAVE_CONFIG_H
  LDFLAGS  = -L/usr/local/Cellar/freetype/2.5.3_1/lib -L/usr/local/lib
  LIBS     = -lwebp -lfreetype -ljpeg -lpng16 -lbz2 -lxml2 -lz -lm -lpthread
```

####测试安装是否成功
```
[wuxuelian@wuxuelianmaccom:~/study/github/graphicsmagick]$ gm convert data/god.jpg god.webp
[wuxuelian@wuxuelianmaccom:~/study/github/graphicsmagick]$ exiftool god.webp 
ExifTool Version Number         : 9.72
File Name                       : god.webp
Directory                       : .
File Size                       : 29 kB
File Modification Date/Time     : 2014:10:23 17:28:08+08:00
File Access Date/Time           : 2014:10:23 16:52:01+08:00
File Inode Change Date/Time     : 2014:10:23 17:28:08+08:00
File Permissions                : rw-r--r--
File Type                       : WEBP
MIME Type                       : image/webp
VP8 Version                     : 0 (bicubic reconstruction, normal loop)
Image Width                     : 1024
Horizontal Scale                : 0
Image Height                    : 768
Vertical Scale                  : 0
Image Size                      : 1024x768
```

###其它功能支持
####不支持interlace渐近功能
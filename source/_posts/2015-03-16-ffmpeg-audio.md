---
layout: post
title: "支持各类音频格式的ffmpeg静态编译"
keywords: ["ffmpeg", "audio"]
description: ""
category: "ffmpeg"
tags: ["ffmpeg"]
date: 2015-03-16
comments: true
---

## 目录
 <div id="wmd-preview-section-24" class="wmd-preview-section preview-content">

</div><div id="wmd-preview-section-11400" class="wmd-preview-section preview-content">

<div><div class="toc"><div class="toc">
<ul>
<li><a href="#概述">概述</a></li>
<li><a href="#依赖库安装">依赖库安装</a><ul>
<li><a href="#yasm130">yasm1.3.0</a></li>
<li><a href="#安装mp3依赖库">安装mp3依赖库</a></li>
<li><a href="#error-libopencoreamrnb-not-found">ERROR: libopencore_amrnb not found</a></li>
<li><a href="#error-libvoamrwbenc-not-found">ERROR: libvo_amrwbenc not found</a></li>
<li><a href="#error-libwavpack-not-found">ERROR: libwavpack not found</a></li>
<li><a href="#error-libaacplus-200-not-found">ERROR: libaacplus &gt;= 2.0.0 not found</a></li>
<li><a href="#error-libfdkaac-not-found">ERROR: libfdk_aac not found</a></li>
<li><a href="#error-libvoaacenc-not-found">ERROR: libvo_aacenc not found</a></li>
</ul>
</li>
<li><a href="#编译安装ffmpeg">编译安装ffmpeg</a><ul>
<li><a href="#常用编译方式小结">常用编译方式小结</a></li>
<li><a href="#支持vorbis">支持vorbis</a></li>
<li><a href="#支持wav">支持wav</a></li>
<li><a href="#支持aac">支持aac</a></li>
<li><a href="#支持mp2">支持mp2</a></li>
<li><a href="#flac-支持">flac 支持</a></li>
<li><a href="#支持-ac3">支持 ac3</a></li>
<li><a href="#支持wmawmv">支持wma/wmv</a></li>
<li><a href="#本次编译涉及所有配置项">本次编译涉及所有配置项</a></li>
</ul>
</li>
</ul>
</li>
</ul>
</div>
</div>
</div></div>


## 概述

本文将详细介绍编译安装ffmpeg,该ffmpeg将支持目前业界各主流音频格式，主要功能为支持mp2，mp3，flac，vorbis，wav，aac，amr，ac3，wma，wmv格式转为mp3/aac/amr。

## 依赖库安装

### yasm1.3.0

编译安装 [yasm-1.3.0.tar.gz](http://www.tortall.net/projects/yasm/releases/yasm-1.3.0.tar.gz) 

### 安装mp3依赖库
   错误
   
   ```
   ERROR: libmp3lame >= 3.98.3 not found
   ```
  
  安装：
  
  ```
  libmp3lame-dev - MP3 encoding library (development)
  apt-get install   libmp3lame-dev
  ```

### ERROR: libopencore_amrnb not found

```
sudo apt-get install libx264-dev libxvidcore-dev libopencore-amrwb-dev libopencore-amrnb-dev libfaad-dev libfaac-dev libmp3lame-dev \
libtwolame-dev liba52-0.7.4-dev libcddb2-dev libcdaudio-dev libcdio-cdda-dev libvorbis-dev libopenjpeg-dev
```

### ERROR: libvo_amrwbenc not found

http://sourceforge.net/projects/opencore-amr/files/vo-amrwbenc/

### ERROR: libwavpack not found

 ```
 sudo apt-get install libwavpack-dev
 ```

### ERROR: libaacplus >= 2.0.0 not found

[ffmpeg官网解决方案](https://trac.ffmpeg.org/wiki/How%20to%20quickly%20compile%20libaacplus)

```
# apt-get install libfftw3-dev pkg-config autoconf automake libtool unzip
$ wget http://tipok.org.ua/downloads/media/aacplus/libaacplus/libaacplus-2.0.2.tar.gz
$ tar -xzf libaacplus-2.0.2.tar.gz
$ cd libaacplus-2.0.2
$ ./autogen.sh --enable-shared --enable-static
$ make
# make install
# ldconfig
```

### ERROR: libfdk_aac not found

编译安装[libfdk_aac](http://sourceforge.net/projects/opencore-amr/?source=directory)

### ERROR: libvo_aacenc not found

编译安装[vo-aacenc-0.1.2.tar.gz](http://sourceforge.net/projects/opencore-amr/files/vo-aacenc/vo-aacenc-0.1.2.tar.gz/download)

## 编译安装ffmpeg

### 常用编译方式小结

1.编译时设置通用参数

```
./configure \
    --extra-cflags='-I/usr/include -static' \
    --extra-ldflags='-I/usr/lib -static' \
    --disable-debug \
    --disable-shared \
    --enable-static \
    --enable-gpl \
    --enable-libmp3lame \
    --enable-nonfree \
    --disable-logging \
    --disable-avdevice \
    --disable-swscale \
    --disable-postproc \
    --disable-dxva2 \
    --disable-vaapi \
    --disable-vda \
    --disable-vdpau \
    --disable-everything \
    --disable-runtime-cpudetect \
    --disable-swscale-alpha \
    --disable-ffplay \
    --disable-ffprobe \
    --disable-ffserver \
    --disable-doc \
    --disable-htmlpages \
    --disable-manpages \
    --disable-podpages \
    --disable-txtpages \
    --enable-protocol=file \
    --enable-protocol=pipe \
    --enable-protocol=http \
    --enable-protocol=https \
    --enable-filter=aresample \
```

2.从第一步生成的config.h中，grep 想要安装的格式关键字，如想要安装mp3

```
fun@ubuntu:~/software/ffmpeg-2.2.2$ grep MP3 config.h 
#define CONFIG_LIBMP3LAME 1
#define CONFIG_MP3_HEADER_DECOMPRESS_BSF 0
#define CONFIG_MP3_DECODER 1
#define CONFIG_MP3FLOAT_DECODER 0
#define CONFIG_MP3ADU_DECODER 0
#define CONFIG_MP3ADUFLOAT_DECODER 0
#define CONFIG_MP3ON4_DECODER 0
#define CONFIG_MP3ON4FLOAT_DECODER 0
#define CONFIG_MP3_DEMUXER 1
#define CONFIG_LIBMP3LAME_ENCODER 1
#define CONFIG_MP3_MUXER 1
```

3.设置诸如encoder,decoder,muxer,demuxer对应项

```
   --enable-libmp3lame \
   --enable-decoder=mp3 \
    --enable-demuxer=mp3 \
    --enable-muxer=mp3 \
    --enable-encoder=libmp3lame \
```


### 支持vorbis

**编译参数**

```
    --enable-libvorbis \
    --enable-parser=vorbis \
    --enable-encoder=vorbis \
    --enable-decoder=vorbis \
    --enable-encoder=libvorbis \
    --enable-decoder=libvorbis \
    --enable-muxer=ogg \
    --enable-demuxer=ogg \
```

**测试成功**

```
fun@ubuntu:~/software/ffmpeg-2.2.2$ ./ffmpeg -i test.mp3 test.ogg
ffmpeg version 2.2.2 Copyright (c) 2000-2014 the FFmpeg developers
  built on Mar 14 2015 01:24:30 with gcc 4.9.1 (Ubuntu 4.9.1-16ubuntu6)
Input #0, mp3, from 'test.mp3':
  Metadata:
    encoder         : Lavf55.33.100
  Duration: 00:00:10.22, start: 0.138125, bitrate: 8 kb/s
    Stream #0:0: Audio: mp3, 8000 Hz, mono, s16p, 8 kb/s
Output #0, ogg, to 'test.ogg':
  Metadata:
    encoder         : Lavf55.33.100
    Stream #0:0: Audio: vorbis (libvorbis), 8000 Hz, mono, fltp
    Metadata:
      encoder         : Lavf55.33.100
Stream mapping:
  Stream #0:0 -> #0:0 (mp3 -> libvorbis)
Press [q] to stop, [?] for help
size=      25kB time=00:00:10.08 bitrate=  20.3kbits/s    
video:0kB audio:22kB subtitle:0 data:0 global headers:3kB muxing overhead 2.472099%
fun@ubuntu:~/software/ffmpeg-2.2.2$ ./ffmpeg -i  test.ogg ogg.mp3
Input #0, ogg, from 'test.ogg':
  Duration: 00:00:10.09, start: 0.000000, bitrate: 20 kb/s
    Stream #0:0: Audio: vorbis, 8000 Hz, mono, fltp, 22 kb/s
    Metadata:
      ENCODER         : Lavf55.33.100
Output #0, mp3, to 'ogg.mp3':
  Metadata:
    TSSE            : Lavf55.33.100
    Stream #0:0: Audio: mp3 (libmp3lame), 8000 Hz, mono, fltp
    Metadata:
      ENCODER         : Lavf55.33.100
Stream mapping:
  Stream #0:0 -> #0:0 (vorbis -> libmp3lame)
Press [q] to stop, [?] for help
size=      10kB time=00:00:10.15 bitrate=   8.3kbits/s    
video:0kB audio:10kB subtitle:0 data:0 global headers:0kB muxing overhead 2.534965%
```

### 支持wav

**编译**

```
 --enable-libwavpack \
    --enable-muxer=wav \
    --enable-demuxer=wav \
    --enable-decoder=wavpack \
    --enable-encoder=wavpack \
    --enable-decoder=wav \
    --enable-encoder=wav \
    --enable-encoder=pcm_s16le \
    --enable-decoder=pcm_s16le \
```

**测试成功**

```
fun@ubuntu:~/software/ffmpeg-2.2.2$ ./ffmpeg -i test.wav wav.mp3
Guessed Channel Layout for  Input Stream #0.0 : mono
Input #0, wav, from 'test.wav':
  Metadata:
    encoder         : Lavf55.33.100
  Duration: 00:00:10.09, bitrate: 128 kb/s
    Stream #0:0: Audio: pcm_s16le ([1][0][0][0] / 0x0001), 8000 Hz, mono, s16, 128 kb/s
Output #0, mp3, to 'wav.mp3':
  Metadata:
    TSSE            : Lavf55.33.100
    Stream #0:0: Audio: mp3 (libmp3lame), 8000 Hz, mono, s16p
Stream mapping:
  Stream #0:0 -> #0:0 (pcm_s16le -> libmp3lame)
Press [q] to stop, [?] for help
[libmp3lame @ 0x29fe940] Trying to remove 576 samples, but the queue is empty
size=      10kB time=00:00:10.15 bitrate=   8.3kbits/s    
video:0kB audio:10kB subtitle:0 data:0 global headers:0kB muxing overhead 2.534965%
fun@ubuntu:~/software/ffmpeg-2.2.2$ ./ffmpeg -i test.mp3 haha.wav
Input #0, mp3, from 'test.mp3':
  Metadata:
    encoder         : Lavf55.33.100
  Duration: 00:00:10.22, start: 0.138125, bitrate: 8 kb/s
    Stream #0:0: Audio: mp3, 8000 Hz, mono, s16p, 8 kb/s
File 'haha.wav' already exists. Overwrite ? [y/N] y
Output #0, wav, to 'haha.wav':
  Metadata:
    ISFT            : Lavf55.33.100
    Stream #0:0: Audio: pcm_s16le ([1][0][0][0] / 0x0001), 8000 Hz, mono, s16, 128 kb/s
Stream mapping:
  Stream #0:0 -> #0:0 (mp3 -> pcm_s16le)
Press [q] to stop, [?] for help
size=     158kB time=00:00:10.08 bitrate= 128.1kbits/s    
video:0kB audio:158kB subtitle:0 data:0 global headers:0kB muxing overhead 0.049574%
```

部分从wav转mp3,遇到一下错误：

```
Input #0, wav, from '../resource/flac/1.wav':
  Duration: 00:09:05.67, bitrate: 96 kb/s
    Stream #0:0: Audio: pcm_u8 ([1][0][0][0] / 0x0001), 12000 Hz, mono, 96 kb/s
[abuffer @ 0x1a3b520] Unable to parse option value "(null)" as sample format
    Last message repeated 1 times
[abuffer @ 0x1a3b520] Error setting option sample_fmt to value (null).
[graph 0 input from stream 0:0 @ 0x1a4fea0] Error applying options to the filter.
Error opening filters!
Conversion failed!
```


解决方法，加上编译参数

```
   --enable-encoder=pcm_u8 \
    --enable-decoder=pcm_u8 \
    --enable-muxer=pcm_u8 \
    --enable-demuxer=pcm_u8 \
```


### 支持aac
**编译**

```
     --enable-libvo-aacenc \
    --enable-libfdk_aac \
    --enable-libfaac \
    --enable-parser=aac \
    --enable-encoder=aac \
    --enable-decoder=aac \
    --enable-encoder=libfaac \
    --enable-encoder=libvo_aacenc \
    --enable-encoder=libaacplus \
    --enable-encoder=libfdk_aac \
    --enable-decoder=libfdk_aac\
    --enable-demuxer=aac \
    --enable-muxer=adts \
```

**测试结果**

```
 ./ffmpeg -i test.mp3 haha.aac
Input #0, mp3, from 'test.mp3':
  Metadata:
    encoder         : Lavf55.33.100
  Duration: 00:00:10.22, start: 0.138125, bitrate: 8 kb/s
    Stream #0:0: Audio: mp3, 8000 Hz, mono, s16p, 8 kb/s
Output #0, adts, to 'haha.aac':
  Metadata:
    encoder         : Lavf55.33.100
    Stream #0:0: Audio: aac (libfdk_aac), 8000 Hz, mono, s16, 17 kb/s
Stream mapping:
  Stream #0:0 -> #0:0 (mp3 -> libfdk_aac)
Press [q] to stop, [?] for help
size=      22kB time=00:00:10.11 bitrate=  18.0kbits/s    
video:0kB audio:22kB subtitle:0 data:0 global headers:0kB muxing overhead 0.000000%
fun@ubuntu:~/software/ffmpeg-2.2.2$ ./ffmpeg -i haha.aac aac.mp3
[aac @ 0x15abd80] Estimating duration from bitrate, this may be inaccurate
Input #0, aac, from 'haha.aac':
  Duration: 00:00:10.86, bitrate: 16 kb/s
    Stream #0:0: Audio: aac, 8000 Hz, mono, fltp, 16 kb/s
File 'aac.mp3' already exists. Overwrite ? [y/N] y
Output #0, mp3, to 'aac.mp3':
  Metadata:
    TSSE            : Lavf55.33.100
    Stream #0:0: Audio: mp3 (libmp3lame), 8000 Hz, mono, fltp
Stream mapping:
  Stream #0:0 -> #0:0 (aac -> libmp3lame)
Press [q] to stop, [?] for help
size=      11kB time=00:00:10.37 bitrate=   8.3kbits/s    
video:0kB audio:10kB subtitle:0 data:0 global headers:0kB muxing overhead 2.482877%
```




### 支持mp2

**编译**

```
    --enable-encoder=mp2 \
    --enable-decoder=mp2 \
    --enable-muxer=mp2 \
    --enable-decoder=mp2float \
    --enable-encoder=mp2fixed \
```

**测试通过**

```
fun@ubuntu:~/software/ffmpeg-2.2.2$ ./ffmpeg -i test.mp3 test.mp2
Input #0, mp3, from 'test.mp3':
  Metadata:
    encoder         : Lavf55.33.100
  Duration: 00:00:10.22, start: 0.138125, bitrate: 8 kb/s
    Stream #0:0: Audio: mp3, 8000 Hz, mono, s16p, 8 kb/s
Output #0, mp2, to 'test.mp2':
  Metadata:
    encoder         : Lavf55.33.100
    Stream #0:0: Audio: mp2, 16000 Hz, mono, s16, 128 kb/s
Stream mapping:
  Stream #0:0 -> #0:0 (mp3 -> mp2)
Press [q] to stop, [?] for help
size=     159kB time=00:00:10.12 bitrate= 128.4kbits/s    
video:0kB audio:159kB subtitle:0 data:0 global headers:0kB muxing overhead 0.000000%
fun@ubuntu:~/software/ffmpeg-2.2.2$ ./ffmpeg -i test.mp2 mp2.mp3
[mp3 @ 0x1b77d80] Estimating duration from bitrate, this may be inaccurate
Input #0, mp3, from 'test.mp2':
  Duration: 00:00:10.29, start: 0.000000, bitrate: 126 kb/s
    Stream #0:0: Audio: mp2, 16000 Hz, mono, s16p, 126 kb/s
File 'mp2.mp3' already exists. Overwrite ? [y/N] y
Output #0, mp3, to 'mp2.mp3':
  Metadata:
    TSSE            : Lavf55.33.100
    Stream #0:0: Audio: mp3 (libmp3lame), 16000 Hz, mono, s16p
Stream mapping:
  Stream #0:0 -> #0:0 (mp2 -> libmp3lame)
Press [q] to stop, [?] for help
size=      30kB time=00:00:10.15 bitrate=  24.3kbits/s    
video:0kB audio:30kB subtitle:0 data:0 global headers:0kB muxing overhead 0.733568%
```

### flac 支持
**编译**

```
   --enable-encoder=flac \
    --enable-decoder=flac \
    --enable-demuxer=flac \
    --enable-muxer=flac \
    --enable-parser=flac \
```

**测试通过**

```
fun@ubuntu:~/software/ffmpeg-2.2.2$ ./ffmpeg -i test.mp3 test.flac
ffmpeg version 2.2.2 Copyright (c) 2000-2014 the FFmpeg developers
Input #0, mp3, from 'test.mp3':
  Metadata:
    encoder         : Lavf55.33.100
  Duration: 00:00:10.30, start: 0.138125, bitrate: 8 kb/s
    Stream #0:0: Audio: mp3, 8000 Hz, mono, s16p, 8 kb/s
Output #0, flac, to 'test.flac':
  Metadata:
    encoder         : Lavf55.33.100
    Stream #0:0: Audio: flac, 8000 Hz, mono, s16, 128 kb/s
Stream mapping:
  Stream #0:0 -> #0:0 (mp3 -> flac)
Press [q] to stop, [?] for help
size=      92kB time=00:00:10.22 bitrate=  73.4kbits/s    
video:0kB audio:84kB subtitle:0 data:0 global headers:0kB muxing overhead 9.643538%
fun@ubuntu:~/software/ffmpeg-2.2.2$ ./ffmpeg -i test.flac flac.mp3
Input #0, flac, from 'test.flac':
  Metadata:
    ENCODER         : Lavf55.33.100
  Duration: 00:00:10.16, bitrate: 73 kb/s
    Stream #0:0: Audio: flac, 8000 Hz, mono, s16
Output #0, mp3, to 'flac.mp3':
  Metadata:
    TSSE            : Lavf55.33.100
    Stream #0:0: Audio: mp3 (libmp3lame), 8000 Hz, mono, s16p
Stream mapping:
  Stream #0:0 -> #0:0 (flac -> libmp3lame)
Press [q] to stop, [?] for help
[libmp3lame @ 0x1a913a0] Trying to remove 576 samples, but the queue is empty
size=      10kB time=00:00:10.22 bitrate=   8.3kbits/s    
video:0kB audio:10kB subtitle:0 data:0 global headers:0kB muxing overhead 2.517361%
```

部分flac转mp3失败的原因之一是ffmpeg没有将图片格式编在里面的缘故，
在ffmpeg编译时添加以下参数

```
    --enable-encoder=jpeg2000 \
    --enable-encoder=mjpeg \
    --enable-encoder=ljpeg \
    --enable-encoder=jpegls \
    --enable-decoder=jpeg2000 \
    --enable-decoder=jpegls \
    --enable-decoder=mjpeg \
    --enable-decoder=mjpegb \
    --enable-muxer=mjpeg \
    --enable-demuxer=mjpeg \
    --enable-encoder=png \
    --enable-decoder=png \
    --enable-parser=png \
```

加入图片支持后，以上转码依旧报错

```
Input #0, flac, from 'b1.flac':
  Duration: 00:08:32.31, bitrate: 871 kb/s
    Stream #0:0: Audio: flac, 44100 Hz, stereo, s16
    Stream #0:1: Video: mjpeg, yuvj420p(pc), 542x475 [SAR 96:96 DAR 542:475], 90k tbr, 90k tbn, 90k tbc
    Metadata:
      comment         : Cover (front)
File 'test.mp3' already exists. Overwrite ? [y/N] y
'scale' filter not present, cannot convert pixel formats.
Error opening filters!
Conversion failed!
```

解决方法：
编译时添加scale的支持

```
   --enable-swscale \
    --enable-swscale-alpha \
    --enable-filter=scale \
```



### 支持 ac3

**编译**

```
    --enable-encoder=ac3 \
    --enable-decoder=ac3 \
    --enable-encoder=ac3_fixed\
    --enable-decoder=atrac3 \
    --enable-decoder=atrac3p \
    --enable-encoder=eac3 \
    --enable-decoder=eac3 \
    --enable-muxer=ac3 \
    --enable-demuxer=ac3 \
    --enable-muxer=eac3 \
    --enable-demuxer=eac3 \
```

**测试通过**

```
./ffmpeg -i test.mp3 test.ac3
Input #0, mp3, from 'test.mp3':
  Metadata:
    encoder         : Lavf55.33.100
  Duration: 00:00:10.30, start: 0.138125, bitrate: 8 kb/s
    Stream #0:0: Audio: mp3, 8000 Hz, mono, s16p, 8 kb/s
Output #0, ac3, to 'test.ac3':
  Metadata:
    encoder         : Lavf55.33.100
    Stream #0:0: Audio: ac3, 8000 Hz, mono, fltp, 96 kb/s
Stream mapping:
  Stream #0:0 -> #0:0 (mp3 -> ac3)
Press [q] to stop, [?] for help
size=     119kB time=00:00:10.14 bitrate=  96.3kbits/s    
video:0kB audio:119kB subtitle:0 data:0 global headers:0kB muxing overhead 0.000000%
fun@ubuntu:~/software/ffmpeg-2.2.2$ ./ffmpeg -i test.ac3 ac3.mp3
[ac3 @ 0x15a7d80] Estimating duration from bitrate, this may be inaccurate
Input #0, ac3, from 'test.ac3':
  Duration: 00:00:10.18, start: 0.000000, bitrate: 96 kb/s
    Stream #0:0: Audio: ac3, 8000 Hz, mono, fltp, 96 kb/s
Output #0, mp3, to 'ac3.mp3':
  Metadata:
    TSSE            : Lavf55.33.100
    Stream #0:0: Audio: mp3 (libmp3lame), 8000 Hz, mono, fltp
Stream mapping:
  Stream #0:0 -> #0:0 (ac3 -> libmp3lame)
Press [q] to stop, [?] for help
size=      10kB time=00:00:10.22 bitrate=   8.3kbits/s    
video:0kB audio:10kB subtitle:0 data:0 global headers:0kB muxing overhead 2.517361%
```

### 支持wma/wmv

编译参数

```
 --enable-decoder=wmalossless \
    --enable-decoder=wmapro \
    --enable-encoder=wmav1 \
    --enable-decoder=wmav1 \
    --enable-encoder=wmav2 \
    --enable-decoder=wmav2 \
    --enable-decoder=wmavoice \
    --enable-demuxer=xwma \
    --enable-demuxer=avi \
    --enable-muxer=avi \
    --enable-demuxer=asf \
    --enable-muxer=asf \
    --enable-encoder=wmv1 \
    --enable-decoder=wmv1 \
    --enable-encoder=wmv2 \
    --enable-decoder=wmv2 \
    --enable-decoder=wmv3 \
    --enable-decoder=wmv3_crystalhd \
    --enable-decoder=wmv3_vdpau \
    --enable-decoder=wmv3image \
```

### 本次编译涉及所有配置项

```
./configure \
    --extra-cflags='-I/usr/include -static' \
    --extra-ldflags='-I/usr/lib -static' \
    --disable-debug \
    --disable-shared \
    --enable-static \
    --enable-gpl \
    --enable-libmp3lame \
    --enable-nonfree \
    --disable-logging \
    --disable-avdevice \
    --disable-swscale \
    --disable-postproc \
    --disable-dxva2 \
    --disable-vaapi \
    --disable-vda \
    --disable-vdpau \
    --disable-everything \
    --disable-runtime-cpudetect \
    --disable-swscale-alpha \
    --disable-ffplay \
    --disable-ffprobe \
    --disable-ffserver \
    --disable-doc \
    --disable-htmlpages \
    --disable-manpages \
    --disable-podpages \
    --disable-txtpages \
    --enable-protocol=file \
    --enable-protocol=pipe \
    --enable-protocol=http \
    --enable-protocol=https \
    --enable-filter=aresample \
   --enable-decoder=mp3 \
    --enable-demuxer=mp3 \
    --enable-parser=mpegaudio \
    --enable-muxer=mp3 \
    --enable-encoder=libmp3lame \
	--enable-version3 \
    --enable-libvo-aacenc \
    --enable-libfdk_aac \
    --enable-libfdk-aac \
    --enable-libfaac \
    --enable-parser=aac \
    --enable-encoder=aac \
    --enable-decoder=aac \
    --enable-encoder=libfaac \
    --enable-encoder=libvo_aacenc \
    --enable-encoder=libaacplus \
    --enable-encoder=libfdk_aac \
    --enable-decoder=libfdk_aac\
	--enable-demuxer=aac \
    --enable-muxer=adts \
    --enable-libopencore-amrnb \
	--enable-libopencore-amrwb \
	--enable-libvo_amrwbenc \
    --enable-encoder=libvo_amrwbenc \
    --enable-decoder=libopencore_amrnb \
	--enable-encoder=libopencore_amrnb \
    --enable-decoder=libopencore_amrwb \
    --enable-decoder=amrnb \
    --enable-decoder=amrwb \
	--enable-muxer=amr \
    --enable-demuxer=amr \
    --enable-libwavpack \
    --enable-muxer=wav \
    --enable-demuxer=wav \
    --enable-decoder=wavpack \
    --enable-encoder=wavpack \
    --enable-encoder=pcm_s16le \
    --enable-decoder=pcm_s16le \
    --enable-libvorbis \
    --enable-parser=vorbis \
    --enable-encoder=vorbis \
    --enable-decoder=vorbis \
    --enable-encoder=libvorbis \
    --enable-decoder=libvorbis \
    --enable-muxer=ogg \
    --enable-demuxer=ogg \
    --enable-decoder=mp1float \
    --enable-decoder=mp1 \
    --enable-encoder=mp2 \
    --enable-decoder=mp2 \
    --enable-muxer=mp2 \
    --enable-decoder=mp2float \
    --enable-encoder=mp2fixed \
    --enable-encoder=flac \
    --enable-decoder=flac \
    --enable-demuxer=flac \
    --enable-muxer=flac \
    --enable-parser=flac \
    --enable-encoder=ac3 \
    --enable-decoder=ac3 \
    --enable-encoder=ac3_fixed\
    --enable-decoder=atrac3 \
    --enable-decoder=atrac3p \
    --enable-encoder=eac3 \
    --enable-decoder=eac3 \
    --enable-muxer=ac3 \
    --enable-demuxer=ac3 \
    --enable-muxer=eac3 \
    --enable-demuxer=eac3 \
    --enable-decoder=wmalossless \
    --enable-decoder=wmapro \
    --enable-encoder=wmav1 \
    --enable-decoder=wmav1 \
    --enable-encoder=wmav2 \
    --enable-decoder=wmav2 \
    --enable-decoder=wmavoice \
    --enable-demuxer=xwma \
    --enable-demuxer=avi \
    --enable-muxer=avi \
    --enable-demuxer=asf \
    --enable-muxer=asf \
    --enable-encoder=wmv1 \
    --enable-decoder=wmv1 \
    --enable-encoder=wmv2 \
    --enable-decoder=wmv2 \
    --enable-decoder=wmv3 \
    --enable-decoder=wmv3_crystalhd \
    --enable-decoder=wmv3_vdpau \
    --enable-decoder=wmv3image \
        --enable-encoder=jpeg2000 \
    --enable-encoder=mjpeg \
    --enable-encoder=ljpeg \
    --enable-encoder=jpegls \
    --enable-decoder=jpeg2000 \
    --enable-decoder=jpegls \
    --enable-decoder=mjpeg \
    --enable-decoder=mjpegb \
    --enable-muxer=mjpeg \
    --enable-demuxer=mjpeg \
    --enable-encoder=png \
    --enable-decoder=png \
    --enable-parser=png \
    --enable-swscale \
    --enable-swscale-alpha \
    --enable-filter=scale \
    --enable-encoder=pcm_u8 \
    --enable-decoder=pcm_u8 \
    --enable-muxer=pcm_u8 \
    --enable-demuxer=pcm_u8 \
    --enable-small \

```

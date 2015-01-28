---
layout: post
title: "使用GraphicsMagick做文字水印"
keywords: ["GraphicsMagick", "Annotate","CompositeImage"]
description: "text watermark with graphicsmagick"
category: "GraphicsMagick"
tags: ["GraphicsMagick"]

---

##使用graphicMagick做图片文字水印
使用GraphicMagick给图片打文字水印，
除文字内容，字体，字体大小等基本属性外，要求可调以下参数：
1.指定文字GravityType，其中Gravity为以下九宫格：

```
NorthWest     |     North      |     NorthEast
              |                |    
              |                |    
--------------+----------------+--------------
              |                |    
West          |     Center     |          East 
              |                |    
--------------+----------------+--------------
              |                |    
              |                |    
SouthWest     |     South      |     SouthEast
```

2.可指定文字边距，包括横轴边距dx,中轴边距 dy。

3.可指定文字的透明度

##分析
###水印位置
水印位置由Gravity和dx,dy共同决定，其中Annotate中可直接指定Gravity，但dx,dy没有直接的参数可以设置。
###水印透明度
文字水印Annotate无法指定文字透明度

##方案
介于以上分析，无法直接使用AnnotateImage来实现，设计方案如下：

###Step 1. 构造水印画布

画一张与原图image同样大小的透明图片textImg,作为文字水印的画布

关键函数：

| 函数        | 说明           | 其它  |
| ------------- |:-------------:| -----:|
|[ConstituteImage](http://www.graphicsmagick.org/api/constitute.html#constituteimage)|构造图片|..|
|[SetImage](http://www.graphicsmagick.org/api/image.html#setimage)|设置图片透明度|..|


```

/**
 * Constitute a transparent Image with nCloumns and nRows
 * @param:img,pointer to save the dest image 
 * @param:nColumns,columns of the dest image 
 * @param:nRows,rows of the dest image 
 * return true with transparent image in img,return false on failed.
 */
bool constituteNewImage(Image * & img, int nColumns, int nRows)
{
    ExceptionInfo exception;
    GetExceptionInfo(&exception);

    const size_t pixels_size = nColumns * nRows * sizeof(Quantum) * strlen(COLOR_MAPPING);
    Quantum * pixels = (Quantum *)MagickMalloc(pixels_size);
    memset((void *) pixels, 0, pixels_size);
    Image * canvasImage = ConstituteImage(nColumns, nRows, COLOR_MAPPING, CharPixel, pixels, &exception);
    //made it transparent
    SetImage(canvasImage,Quantum(MaxRGB));//must
    canvasImage->matte = true;//musta
    MagickFree(pixels);
    DestroyExceptionInfo(&exception);

    if (img != NULL)
        DestroyImageList(img);
    img = canvasImage;
    return true;
}
```

### Step2 : 制作文字图片

使用Annotate函数在textImg图上打上文字，并指定Annotate为Center，即写在正中。

```
/**
 * Draw Text on the center of the image 
 *@param:image,image to annotate
 *@param:text,text to annotate
 *@param:font_pointsize,fontsize in point
 *@param:fill_color,color of font 
 *return true on success while false on failed
 */
bool DrawText(Image *& image, const char * text, const double font_pointsize,const char * fill_color) 
{
    cout << "text:" << text << endl;
    cout << "color:" << fill_color << endl;

    DrawContext draw_context;
    draw_context = DrawAllocateContext((DrawInfo*)NULL, image);
    DrawSetFillColorString(draw_context, fill_color);

    DrawSetStrokeAntialias(draw_context, 0);
    DrawSetTextAntialias(draw_context, 0);
    DrawSetFont(draw_context, FONT_DEFAULT);
    DrawSetFontSize(draw_context, font_pointsize);
    DrawSetGravity(draw_context, CenterGravity);
    DrawSetTextEncoding(draw_context, "UTF-8");
    DrawAnnotation(draw_context, 0, 0, (const unsigned char *)text);
    unsigned int ret = DrawRender(draw_context);
    DrawDestroyContext(draw_context);
    return ret;
}

```
###Step 3:裁减获得文字logo
分析图textImg的每一个像素，找到包含水印文字的最小矩形所在位置，使用CropImage获得该矩形所在图作为logo。

```
/** 
 * Trim Image
 * remove transparent part around the Text
 * return image with text only
 */
Image * trimImage(Image *& image){
    if(IsGIF(image)) {
       return NULL;
    }
    int y, x;
    int sx,sy,ex,ey;
    register PixelPacket *q;
    RectangleInfo rect;
    ExceptionInfo exception;
    GetExceptionInfo(&exception);
    sx = image->columns;
    sy = image->rows;
    ex = 0;
    ey = 0;

    for (y=0; y < (long) image->rows; y++)
    {
        q=GetImagePixels(image,0,y,image->columns,1);
        if (q == (PixelPacket *) NULL)
            break;
        for (x=0; x < (long) image->columns; x++)
        {
            if(q->opacity != MaxRGB) {
                sx = sx < x ? sx:x;
                sy = sy < y ? sy:y;
                ex = ex > x ? ex:x;
                ey = ey > y ? ey:y;                
            }
            q++;
        }
    }
    cout << "sx:" << sx << ",sy:" << sy << ",ex:" << ex << ",ey:" << ey << endl;
    if(sx >= ex || sy >= ey) { // Too small
        rect.x = 0;
        rect.y = 0;
        rect.width = image->columns;
        rect.height = image->rows;
    } else {
        rect.x = sx;
        rect.y = sy;
        rect.width = ex - sx + 1;
        rect.height = ey - sy + 1;
    }
    cout << "x:" << rect.x << ",y:" << rect.y << ",rect.width:" << rect.width << ",rect.height:" << rect.height << endl;
    Image *cImage = CropImage(image, &rect, &exception);
    cout << "cImage clomnes:" << cImage->columns << ",rows:" << cImage->rows << endl;

    DestroyExceptionInfo(&exception);
    return cImage;
}

```
### Step4:设置文字logo透明度
设置logo图片非透明部分透明度

```

/**
 * set dissolve 
 * @param image 
 * @dissolve:0~100,0 means totally transparent while 100 means opa,q
 * */
MagickPassFail dissolveImage(Image *image,int dissolve){
    int y, x;
    register PixelPacket
        *q;
    for (y=0; y < (long) image->rows; y++)
    {
        q=GetImagePixels(image,0,y,image->columns,1);
        if (q == (PixelPacket *) NULL)
            break;
        for (x=0; x < (long) image->columns; x++)
        {
            if(q->opacity != MaxRGB) {
                q->opacity=(Quantum)
                    (MaxRGB - ((MaxRGB-q->opacity)/100.0*dissolve));
            }
            q++;
        }
        if (!SyncImagePixels(image)){
            return MagickFail;
        }
    }
    return MagickPass;
}

```
Step 5:将logo打到原图对应位置
 
根据gravity算出logo图该落的偏移位置x_offset,y_offset,以CompositeImage的方式将logo图片拼接到原图上。

logo 偏移量计算

| Gravity        | x_offset           | y_offset  |
| ------------- |:-------------:| -----:|
|NorthWestGravity | dx|dy |
|NorthGravity|image.cols/2 - logo.cols/2 + dx|dy|
|NorthEastGravity|image.cols-dx-logo.cols|dy|
|WestGravity|dx|image.rows/2 - logo.rows/2 + dy|
|CenterGravity|image.colns/2 - logo.cols/2 + dx|image.rows/2 - logo.rows/2 + dy|
|EastGravity|image.cols - dx - logo.cols|image.rows/2 - logo.rows/2 + dy|
|SouthWestGravity|dx|image.rows - logo.rows - dy|
|SouthGravity|image.cols/2 - logo.cols/2 + dx|image.rows - logo.rows - dy|
|SouthEastGravity|image.cols - dx - logo.cols|image.rows - logo.rows - dy|

```
/**
 *Composite Image 
 *@param image:source image 
 *@param logo:logo image 
 *@param gravity,
 *|NorthWestGravity |NorthGravity   |NorthEastGravity   |
 *|WestGravity      |CenterGravity  |EastGravity        |
 *|SouthEastGravity |SouthGravity   |SouthEastGravity   |
 *@param dx:offset on x
 *@param dy:offset on y
 * return true on success,false on failed 
 */
bool compositeImage(Image *image,Image *logo,GravityType gravity,long dx,long dy) {
   int x_offset,y_offset;

   switch(gravity){
       case NorthWestGravity:{
           x_offset = dx;
           y_offset = dy;
           break;
        }
       case NorthGravity:{
           x_offset = image->columns/2 - logo->columns/2 + dx;
           y_offset = dy;
           break;
       }
       case NorthEastGravity: {
           x_offset = image->columns-dx-logo->columns;
           y_offset = dy;
           break;
       }
       case WestGravity:{
           x_offset = dx;
           y_offset = image->rows/2 - logo->rows/2 + dy;
           break;
       }
       case CenterGravity: {
          x_offset = image->columns/2 - logo->columns/2 + dx;
          y_offset = image->rows/2 - logo->rows/2 + dy;
          break;
       }
       case EastGravity:{
          x_offset = image->columns - dx - logo->columns;       
          y_offset = image->rows/2 - logo->rows/2 + dy;
          break;
       }
     case SouthWestGravity:{
          x_offset = dx;
          y_offset = image->rows - logo->rows - dy;
          break;
       }

       case SouthGravity:
           x_offset = image->columns/2 - logo->columns/2 + dx;
           y_offset = image->rows - logo->rows - dy;
           break;
       
       case SouthEastGravity:{
           x_offset = image->columns - dx - logo->columns;
           y_offset = image->rows - logo->rows - dy;
           break;
       }
       default:{
           cout << "Illegal GravityType:" << gravity;      
           break;
       }
   }

    if(CompositeImage(image, AtopCompositeOp,logo, x_offset, y_offset) != 1) {
          return false;
    }
    return true;
}

```

##完整示例
###代码
[water_mark_txt](https://github.com/AndreMouche/GraphicsMagicUsage/blob/master/water_mark_txt.cpp)

###效果

<img src="https://raw.githubusercontent.com/AndreMouche/GraphicsMagicUsage/master/data/water_mark_txt.jpg" alt="water_mark_txt.jpg" title="water_mark_txt.jpg" width="400" />


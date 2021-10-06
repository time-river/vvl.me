---
title: canvas第三课
date: 2016-07-27 15:01:52
categories: style
tags: [javascript, canvas]
---

慕课网 Canvas 第三课[Canvas玩转图像处理](http://www.imooc.com/learn/476)。

## 总结

### image 相关

```javascript
var img = new Image();   // 创建一个<img>元素
img.src = "myImage.png"; // 设置图片源地址
// 使用load事件来保证不会在图片加载完毕前使用这个图片
img.onload = function(){
  //  执行drawImage语句
}
```

还可以通过 [data: url](#data-url) 方式嵌入图像

### Canvas 相关

---
方法 | 描述 | 参数
--- | --- | ---
*drawImage(image, x, y)* | 绘制图片 | *image* 是 image 或 canvas 对象，*x* 和 *y* 是其在目标 canvas 里的起始坐标
*drawImage(image, x, y, width, height)* | 缩放 Scaling | *width* 与 *height* 用于控制当 canvas 画入时应该缩放的大小
*drawImage(image, sx, sy, sWisth, sHeight, dx, dy, dWidth, dHeight)* | 切片 Slicing | 2-5 定义图片源的切片位置和大小，后四个定义切片的目标显示位置和大小
*createImageData(width, height)* | 创建一个特定尺寸的 ImageData 对象，所有像素被预设为透明黑 |
*getImageData(left, top, width, height)* | 获得一个包含画布场景像素数据的 ImageData 对像 | 矩形四个顶点
*putImageData(myImageData, dx, dy)* | 对场景进行像素数据的写入 |

### 鼠标事件

[耸肩]没找到相关资料，自己明白？比较模糊，那就不写啦。

### Mozilla 文档很不错。

[Image()](https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API/Tutorial/Using_images#获得需要绘制的图片)
[像素操作](https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API/Tutorial/Pixel_manipulation_with_canvas)

## 一些知识

### data:url

* 格式 `data:[<media type>][;base64],<data>`
* [Data URI scheme](https://en.wikipedia.org/wiki/Data_URI_scheme)
* 一个图片[在线转换](https://www.base64-image.de/)

### 像素点

* 图片中，JavaScript 像素级操作，一个像素点是一组 RGBA 值
* 第 i 个像素(pixelData)
  * R = pixelData[4*i+0]
  * G = pixelData[4*i+1]
  * B = pixelData[4*i+2]
  * A = pixelData[4*i+3]
* 第 x 行第 y 列的像素
  * i = x * width + y

## 一些实例

* 插入一个图片
  * [演示](/demo/16-07-27/demo-1.html)
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-27/demo-1.html)
* 图片缩放与水印
  * [演示](/demo/16-07-27/demo-2.html)
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-27/demo-2.html)
* 局部放大
  * `clip()` 探照灯效果
  * [演示](/demo/16-07-27/demo-3.html)
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-27/demo-3.html)
* 滤镜 -- 像素级操作
  * 有个跨域问题，不过可以利用[data-url](#data-url)解决
  * [演示](/demo/16-07-27/demo-4.html)
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-27/demo-4.html)
* [有没有一段代码，让你觉得人类的智慧也可以璀璨无比？](https://www.zhihu.com/question/30262900)
  * [演示](/demo/16-07-27/demo-5.html)
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-27/demo-5.html)

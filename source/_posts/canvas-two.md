---
title: canvas 第二课
date: 2016-07-25 17:39:04
categories: style
tags: [javascript, Canvas]
mathjax: true
---

慕课网 Canvas 第二课[Canvas绘图详解](http://www.imooc.com/learn/185)。

## 总结

### 颜色、样式和阴影

---
属性 | 描述 | 值
--- | --- | ---
*strokeStyle* | 笔触的颜色、渐变或模式 | [CSS 颜色](#CSS-颜色表示法)，渐变色
*fillStyle* | 填充绘画的颜色、渐变或模式 |
*shadowColor* | 阴影的颜色、渐变或模式 | [CSS 颜色](#CSS-颜色表示法)
*shadowBlur* | 阴影的模糊级别 | 有效值，值越大越模糊，0 被忽略
*shadowOffsetX* | 阴影距形状的水平距离 |
*shadowOffsetY* | 阴影距形状的垂直距离 |

---
方法 | 描述 | 值
--- | --- | ---
*createLinearGradient(\\(x_0\\), \\(y_0\\), \\(x_1\\), \\(y_1\\))* | 创建线性渐变（用在画布内容上，不指定渐变范围）| 起点\\((x_0, y_0)\\)，终点\\((x_1, y_1)\\)
*createRadialGradient(\\(x_0\\), \\(y_0\\), \\(r_0\\), \\(x_1\\), \\(y_1\\), \\(r_1\\))* | 创建放射状 / 环形的渐变 |
*addColorStop(offset, color)* | 规定渐变对象的颜色和停止位置 | *offset* 范围 0-1
*createPattern(img/canvas/video, repeat-style)* | 在指定的方向上重复指定的元素 | repetition: *repeat* / *repeat-x* / *repeat-y* / *no-repeat*

### 线条样式

---
属性 | 描述 | 值
--- | --- | ---
*lineCap* | 线条的结束端点样式 | *butt*(Default) / *round* / *square*
*lineJoin* | 两条线相交时，所创建的拐角类型 | *miter*(Default) / *bevel* / *round*
*lineWidth* | 当前线条的宽度 | 有限值
*miter* | 最大[斜切](#斜切)长度 |
*miterLimit* | [斜切](#斜切)的限制比例 |

### 矩形

---
方法 | 描述 | 参数
--- | --- | ---
*rect(x, y, width, height)* | 创建矩形 | 左上角顶点 (x, y)，宽 width，高 height
*strokeRect(x, y, width, height)* | 绘制已定义的路径 |
*fillRect(x, y, width, height)* | 绘制“被填充”的矩形 |
*clearRect(x, y, width, height)* | 在给定的矩形内清除指定的像素 |

### 路径

---
方法 | 描述 | 参数
--- | --- | ---
*stroke()* | 绘制已定义的路径 |
*fill()* | 填充当前绘图（路径） |
*beginPath()* | 起始一条路径，或重置当前路径 |
*closePath()* | 创建从当前回到起始点的路径 |
*moveTo(x, y)* | 使用给定点创建子路径 |
*lineTo(x, y)* | 增加给定的点，用线段连接到前一个 |
*arc(x, y, radius, startAngle, endAngle[, counterclockwise])* | 创建弧/曲线（用于创建圆形或部分圆）| 圆心，半径，起始/结束角度，可选参数--旋转方向
*arcTo(\\(x_1\\), \\(y_1\\), \\(x_2\\), \\(y_2\\), radius)* | 创建两切线之间的弧 / 曲线 | \\(A(x_0, y_0)\\), \\(B(x_1, y_1)\\), \\(C(x_2, y_2)\\)，切线 \\(AB\\), \\(BC\\)
*clip()* | 从原始画布剪切任意形状和尺寸的区域 |
*quadraticCurveTo(cpx, cpy, x, y)* | 二次贝塞尔曲线 |
*bezierCurveTo()* | 三次贝塞尔曲线 |
*isPointInPath(x, y)* | 如果指定的点位于当前路径中，则返回 *true*，否则返回 *false* |

### 二维坐标变换

---
方法 | 描述 | 参数
--- | --- | --
*scale(x, y)* | 展缩 |
*rotate(angle)* | 旋转 |
*translate(x, y)* | 平移 |
*transform(a, b, c, d, e, f)* | 坐标变换 | [二维变换矩阵](#二维变换矩阵)
*setTransform(a, b, c, d, e, f)* | 由原始图形经[二维变换矩阵](#二维变换矩阵)变换 |

### 文本

---
属性 | 描述 | 值
--- | --- | ---
*font* | 文本字体属性 | [CSS 字体样式](#CSS-字体样式)
*textAlign* | 文本对齐方式 | *start*(Default) / *end* / *left* / *right* / *center*
*textBaseline* | 文本基线 | *top* / *middle* / *bottom* / *alphabetic*(Default) / *ideographic* / *hanging*

---
方法 | 描述 | 参数
--- | --- | ---
*strokeText(text, x, y[, maxWidth])* | 绘制文本边缘线 |
*fileText(text, x, y[, maxWidth])* | 绘制被填充的文本 |
*measureText(text)* | 文本度量，属性*width* |

### 图像绘制

---
方法 | 描述
--- | ---
*drawImage()* | 绘制图像、画布、视频

### 合成

---
属性 | 描述 | 值
--- | --- | ---
*globalAlpha* | 绘图的当前 alpha 或透明值 | 范围 0-1
*globalCompositeOperation* | 图层叠加 | *source-over*(Default) / *source-atop* / *source-in* / *source-out* / *destination-over* / *destination-atop* / *destination-in* / *destination-out* / *lighter* / *copy* / *xor*

### 其他

---
方法 | 描述 | 参数
--- | --- | ---
*save()* | 保存当前环境状态 |
*restore()* | 返回之前保存过的环境状态 |
*getContext()* | 2D 绘图，参数为 *2d* |

## 涉及到的知识

### CSS 颜色表示法

* 英文。如 "black"
* 十六进制颜色: #RRGGBB。如 "#000000"，缩写 "#000"
* RGB 颜色: rgb(R, G, B), 参数范围 0-255，或百分比。如 "rgb( 0, 0, 0)"
* RGBA 颜色: rgba(R, G, B, A), A 范围 0-1
* HSL 颜色: hsl(H, S, L)
* HSLA 颜色: hsla(H, S, L, A)
* R: Red, G: Green, B: Blue, A: Alpha(透明度)
* H: Hue(色调，色盘度数，范围0-360), S: Saturation(饱和度，百分比值), L: lightness(亮度，百分比值)
* [CSS 合法颜色值](http://www.w3school.com.cn/cssref/css_colors_legal.asp)

### CSS 字体样式

* font: font-style font-variant font-weight font-size font-family;
* [<字体风格> || <字体变形> || <字体加粗> ]? <字体大小> [ / <行高> ]? <字体类形>
* [css字体样式(Font Style),属性](http://blog.leanote.com/post/278536682@qq.com/aa1bdd26cd9b)

### 斜切

* [HTML 5 canvas miterLimit 属性](http://www.w3school.com.cn/tags/canvas_miterlimit.asp)

### 贝塞尔曲线

* [贝塞尔曲线扫盲](http://www.html-js.com/article/1628)
* [贝塞尔曲线](https://zh.wikipedia.org/wiki/%E8%B2%9D%E8%8C%B2%E6%9B%B2%E7%B7%9A)
* [Canvas Bézier Curve Example](http://tinyurl.com/html5bezier)

### 二维变换矩阵

* 计算机图形学
* a:水平缩放 c:垂直倾斜 e:水平位移
* b:水平倾斜 d:垂直缩放 f:垂直位移
* {% raw %}$
\begin{bmatrix}
a & c & e \\
b & d & f \\
0 & 0 & 1
\end{bmatrix}
${% endraw %}
* [变换矩阵](https://zh.wikipedia.org/wiki/%E5%8F%98%E6%8D%A2%E7%9F%A9%E9%98%B5)

### 非零环绕原则

* [Ch21 非零环绕原则](https://airingursb.gitbooks.io/canvas/content/21.html)

## 效果

* 剪辑区域`clip()` -- 探照灯效果
* 非零环绕原则 -- 剪纸效果

## 一些实例

注意函数的组织形式，使用可选参数的技巧。

* 圆角矩形 -- 2048 方格
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-25/demo-1.html)
  * [演示](/demo/16-07-25/demo-1.html)
* 五角星
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-25/demo-2.html)
  * [演示](/demo/16-07-25/demo-2.html)
* 弯月
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-25/demo-3.html)
  * [演示](/demo/16-07-25/demo-3.html)
* 星空
  * 五角星 + 弯月 + 三次贝塞尔曲线 + 渐变
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-25/demo-4.html)
  * [演示](/demo/16-07-25/demo-4.html)
* *globalCompositeOperation* 属性演示
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-25/demo-5.html)
  * [演示](/demo/16-07-25/demo-5.html)
* Canvas绘图之旅 -- 运动的小球
  * 点击事件
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-25/demo-6.html)
  * [演示](/demo/16-07-25/demo-6.html)

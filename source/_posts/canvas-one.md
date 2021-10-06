---
title: Canvas 第一课
date: 2016-07-22 15:30:37
categories: style
tags: [javascript, canvas]
---

使用 Canvas 制作[绚丽的倒计时效果](http://www.imooc.com/learn/133)。

## 总结

* HTML

```html
<canvas id="canvas></canvas>
```

* JavaScript
  * canvas 画布坐标
  * 获取 canvas 节点
  * 获取上下文绘图环境`context`
  * 使用`context`进行绘制
  * Canvas 绘图，总是先绘制状态，再调用函数绘制
  * 绘制直线
  * 绘制弧线
  * 开始/结束路径，并非成对出现
  * `context`属性 -- 状态的改变
  * 绘制线条，填充图形
  * 画布中矩形区域的刷新
  * `canvas`画布接口
  * 动画函数`setInterval`

```javascript
var canvas = document.getElementById("canvas");
var context = canvas.getContext("2d");
// 使用 context 进行绘制

// 绘制直线
context.moveTo(x, y); // 起点
context.lineTo(x, y); // 终点

// 绘制弧线
context.arc(centerx, centery, radius, startingAngle, endingAngle, anticlockwise = false); // (极座标系) 圆心坐标，起始角度，结束角度，默认逆时针

// 开始/结束路径
context.beginPath();
context.closePath();

// 状态的改变
context.lineWidth // 线条宽度
context.strokeStyle // 线条颜色
context.fillStyle // 填充颜色

context.stroke(); // 绘制线条
context.fill(); // 填充图形

context.clearRect(x, y, width, height);

context.canvas // context上下文的画布

// canvas画布
canvas.width
canvas.height
canvas.getContext("2d");

// Animation
setInterval(
  function(){ //动画函数
    render(); //通常包括渲染&纠正
    update();
  },
  50 //间隔时间毫秒
);
```

## 一些实例

* 检测浏览器是否支持 canvas 标签

```javascript
<canvas>不支持Canvas</canvas> //浏览器不支持Canvas标签，则会显示标签的内容。

var canvas = document.getElementById("canvas");
canvas.getContext("2d"); // 若浏览器不支持，返回 null
```

* 绘制直线/三角形
  * beginPath / moveTo / lineTo / closePath
  * lineWidth / strokeStyle / stroke / fill
  * [演示](/demo/16-07-23/demo-1.html)
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-23/demo-1.html)
* 绘制线条与填充图形下的`beginPath()`与`closePath()`
  * 注意他们的代码的区别
  * [演示](/demo/16-07-23/demo-2.html)
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-23/demo-2.html)
* 七巧板
  * 以上的应用
  * [演示](/demo/16-07-23/demo-3.html)
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-23/assert/tangram/tangram.js)

* 下落的小球
  * 动画函数 `setInterval(func, time)`
  * [演示](/demo/16-07-23/demo-4.html)
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-23/demo-4.html)
* 绚丽的倒计时效果
  * 包含以下几部分
    * 静止时钟
      * [模板源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-23/assert/clock/digit.js)
      * 涉及函数 `render()` `renderDigit()`
    * 动态时钟
      * `update()`
    * 小球动画
      * `updateBalls()`
      * `addBalls()`
    * 性能优化
    * 屏幕自适应
  * [演示](/demo/16-07-23/demo-5.html)
  * [源码](https://github.com/time-river/time-river/blob/master/canvas/16-07-23/assert/clock/countdown.js)

## 来源

[慕课网 绚丽的倒计时效果](http://www.imooc.com/learn/133)

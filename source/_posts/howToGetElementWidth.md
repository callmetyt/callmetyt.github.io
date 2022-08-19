---
title: howToGetElementWidth
date: 2022-08-16 21:42:37
tags:
 - Javascript
 - Web Api
categories:
 - javascript
description: 获取元素width的一些方式
top_image: https://cdn.pixabay.com/photo/2016/06/24/09/18/measurement-1476913_960_720.jpg
cover: /img/cover_10.webp

---
## 前言
- 一实习就没有了更新的动力😭（主要还是太懒了），直到最近完成一个需求时，才想到可以把这次学习到的一点点东西总结成一片博客好好水一水
- 本次需求中有一个核心诉求点，**需要知道一个textNode中某几个文字的宽度**，例子如下：
```html
<span>这是一段文字</span>
<!-- 如何获取“这是一段文字”中“文字”的宽度 -->
```
- 其实可以扩展为**如何获取一个元素的大小（DOMRect）**

## 可行方案
### getBoundingClientRect
- 最常用，最常见，大多数人都应该知道的一个API
- 存在于HTMLElement元素上，调用方式Element.getBoudingClientRect
- 可以获取调用此方法的元素的**DOMRect以及基于当前视窗的位置**
  - 视窗原点指的是页面左上角
  - 所有单位都是**像素（px）**
```typescript
interface DOMRect{
    height: number; // 元素高度
    width: number; // 元素宽度
    left: number; // 元素左边界距离视窗原点的宽度
    top: number; // 元素上边界距离视窗原点的宽度
    x: number; // 同left
    y: number; // 同right
    bottom: number; // 元素下边界距离视窗原点的宽度，height + top
    right: number; // 元素右边界距离视窗原点的宽度，width + left
}
```
- width和盒子模型
  - width的值在标准盒子模型会是border+padding+width
  - 在border-sizing: border-box则会是content width，没有border和padding

### getClientRects
- 实际上是**getBoundingClientRect的扩展版**，特殊处理了一下行内元素
- 简述一下：块级元素和getBoundingClientRect一致，但是**行级元素则会在换行情况下计算为一个新DOMRect**
- 实例
```html
<span>这是一个超长的文本，实际渲染就会造成换行!这是一个超长的文本，实际渲染就会造成换行!这是一个超长的文本，实际渲染就会造成换行!这是一个超长的文本，实际渲染就会造成换行!这是一个超长的文本，实际渲染就会造成换行!这是一个超长的文本，实际渲染就会造成换行!这是一个超长的文本，实际渲染就会造成换行!这是一个超长的文本，实际渲染就会造成换行!</span>
<script>
    const span = document.querySelector("span");
    console.log(span.getClientRects());
</script>
```
  - 代码运行结果

{% asset_img 1.png 代码运行结果 %}

  - 每一行文本对应一个DOMRect
- 在本次需求中本应是最完美的解决方案，但是由于一些原因，需要去获取某几个文字的宽度，并非一个标签，所以需要结合下面这个API使用

### Range
- 一个因为本次需求才知道的Web API，很nice的浏览器兼容性

{% asset_img 2.png caniuse %}

- 可以创建一个Range对象，调用方法设置起点（start）和终点（end），然后就**可以调用上面两个元素API，获取DOMRect**（实际用起来很cool～）
- 示例
  - render函数会根据传入的DOMRect，渲染出一个红色边框
```html
<body>
  <div class="box1">这是box1</div>
  <div class="box2">这是box2</div>
</body>
<script>
  const box1 = document.querySelector('.box1');
  const box2 = document.querySelector('.box2');
  const range = document.createRange(); // 创建Range
  range.setStart(box1, 0); // 设置起点
  range.setEnd(box1, 1); // 设置终点
  render(range.getBoundingClientRect());
</script>
```
  - 代码运行结果

{% asset_img 3.png 代码运行结果 %}

  - 因为box1只有一个children Node（“这是box1”的textNode），所以终点只能设置为1，如果**超过children Node就会报错**
  - 那么当我把终点设置为box2的textNode，会发生什么呢？
```html
<script>
  // ...
  range.setEnd(box2, 1); // 设置终点
</script>
```
  - 运行结果

{% asset_img 4.png 代码运行结果 %}

  - 可以看到**完美囊括了box1和box2**，那么我们反其道而行之，设置box1的textNode会发生什么？
```html
<script>
  // ...
  const box1TextNode = box1.childNodes[0]; // 获取box1的TextNode
  range.setStart(box1TextNode, 0);
  range.setEnd(box1TextNode, 2);
  // ...
</script>
```
  - 代码运行结果

{% asset_img 5.png 代码运行结果 %}

  - 完美达到了我想要的效果！只在**前两个文字（“这是”）上面绘制了边框**，就是说只获取了前两个文字的DOMRect
- 至此，我就找到了Web API所支持的标准解法（当然会有其他hack方式，下文就是一个）

PS：Range也可以调用getClientRects方法，所以遇到换行的字符也可以准确获取width

### measureText
- 这个其实是一种hack方式，measureText其实是canvas的2d上下文的一个api，可以设置文字大小和字体（font-size和font- family）**计算文字宽度**
- 这种方式完全**不需要获取目标文字所在的相关DOM**，你只需要创建一个canvas的绘制上下文即可
```javascript
const ctx = document.createElement('canvas').getContext('2d'); // 创建上下文
ctx.font = '12px serif'; // 设置字体和大小
ctx.measureText('这是'); // 获取文字大小
```
- 值得注意的是，在设置ctx.font的时候，需要设置为当前canvas已加载的字体

# 相关资料
- MDN
  - [getBoundingClientRect](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/getBoundingClientRect)
  - [getClientRects](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/getClientRects)
  - [RangeAPI](https://developer.mozilla.org/zh-CN/docs/Web/API/Range)
  - [measureText](https://developer.mozilla.org/zh-CN/docs/Web/API/CanvasRenderingContext2D/measureText)
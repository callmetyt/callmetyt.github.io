---
title: React Schedule基础学习
date: 2022-05-21 16:53:20
tags:
 - React
 - Javascript
 - concurrent
categories:
 - React
description: 关于React的并发渲染模式的基础理念学习
top_image: https://pic2.zhimg.com/v2-8c3223d9228013e069df6a302e81c38f_720w.png?source=d16d100
cover: /img/cover_7.webp

---
## 前言

- 之前学习React源码过程中，相关文章一直提及React的concurrent mode（并发渲染模式），由此需要先学习一些前置知识
  - **`requestIdleCallback`：由浏览器提供的一个API**，类似于`requestAnimation`，可以再特定时间段中执行代码
  - **然而，上述API由于在实验中，React模拟实现了一个自己的API，不过不妨碍用`requestIdleCallback`去理解React的调度**

## requestIdleCallback

### 官方文档

- [MDN对此的文档](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestIdleCallback)

### 简述

- **`requestIdleCallback`可以让我们在浏览器空闲时间运行某些代码**
- 浏览器一帧所运行的过程如下
  - 熟知的`requestAnimationFrame`会在浏览器解析HTML前执行（**也是动画逻辑最适宜插入的地方**）
  - **`requestIdleCallback`会在浏览器完成一帧的绘制后执行**

{% asset_img 1.png 浏览器一帧所做的事情 %}

- 如果有空闲时间的话，就会去执行`requestIdleCallback`函数中的回调函数

  - 基本例子

    ```javascript
    function someWork(deadline){ // deadline就是requestIdleCallback提供的参数
        console.log('当前帧剩余时间:',deadline.timeRemaining()); // ms
        if(deadline.timeRemaining() > 1){
            console.log('还有剩余时间'); // 当前帧有空闲时间执行代码
        }else if(deadline.didTimeout){
            console.log('超时了！'); // 当前帧没有空闲时间，且已超过timeout，浏览器就会立即执行
        }else{
            requestIdleCallback(someWork,{timeout:1000}); // 当前帧没有空闲时间，且没过timeout，将回调函数放到下一帧去
        }
    }
    requestIdleCallback(someWork,{timeout:1000});
    ```

## React调度——时间分片

### 并发执行

- React之所以使用Fiber，很大程度上是为了实现concurrent mode（并发渲染），为了实现大概这样的效果
  - 利用Fiber这种类树的数据结构，在对每一个Fiber执行完`Work`的时候，可以中断，也就是停止JavaScript的执行，将线程交给浏览器
  
  - **这样就会达到线程在JavaScript执行和浏览器绘制间交替运行——也就是并发执行！**
  
    ```javascript
    // 并发渲染模式下的workLoop函数如下
    function workLoopConcurrent() {
        // Perform work until Scheduler asks us to yield
        while (workInProgress !== null && !shouldYield()) { // shouldYield函数用于查看是否需要把线程归还给浏览器
            performUnitOfWork(workInProgress);
        }
    }
    ```
  
  - `ShouldTield`函数如下：
  
    ```javascript
    function shouldYieldToHost() {
        var timeElapsed = exports.unstable_now() - startTime;
        if (timeElapsed < frameInterval) { // frameInterval 一般是5ms
            // 运行时间很小，不需要yield
            return false;
        }
        // 需要把线程返回给浏览器
        return true;
    }
    ```

### React 实际使用的方式

- React为了兼容性（因为`requestIdleCallback`函数是实验性函数），模拟实现了自己的`requestIdleCallback`函数

- 宏任务无疑是最佳的模拟方式，原因在后面，**React采用了`MessageChannel`方式模拟自己的`requestIdleCallback`函数**

  ```javascript
  const channel = new MessageChannel()
  const port = channel.port2
  
  // 每次 port.postMessage() 调用就会添加一个宏任务
  // 该宏任务为调用 scheduler.scheduleTask 方法
  channel.port1.onmessage = scheduleTask
  
  function scheduleTask(){
      const task = getTask(); // 获取一个任务并执行
      if(!task()){
          port.postMessage(null); // 任务未完成，下一个任务继续
      }
  }
  ```

- 选择宏任务的原因，这里需要注意宏任务和微任务的执行时机

  - **宏任务不会阻塞页面渲染，而微任务会阻塞**

{% asset_img 2.png JS代码去描述浏览器执行 %}

  - 正如图片中所描述的，每一次渲染都会将微任务清空，才会去进行下一步

  - [网上找到的DEMO例子](https://codepen.io/jackluson/pen/eYVgewE)


## 文章推荐

> [2022年了,还不懂requestIdleCallback么?——掘金](https://juejin.cn/post/7061947637167194142)
>
> [React调度（Schedule）原理理解](https://blog.csdn.net/weixin_45654582/article/details/122634214)
>
> [React Scheduler 为什么使用 MessageChannel 实现](https://juejin.cn/post/6953804914715803678)


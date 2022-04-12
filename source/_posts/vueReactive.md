---
title: Vue响应式原理的简单探索
date: 2021-6-4 09:06:01
comments: true
tags:
- Vue
categories:
- Vue
description: Vue响应式原理的实现原理，以及用JavaScript书写一个简单的响应式数据
cover: /img/cover_2.webp
top_image: https://pic2.zhimg.com/v2-db7221c0d7ca752b2f88d7ca94939976_1440w.jpg?source=172ae18b

---
## 引入

_Vue.js 的核心是一个允许采用简洁的模板语法来声明式地将数据渲染进 DOM 的系统_——Vue官方文档

- 第一个例子（只保留核心代码）

  ```html
  <div id="app">
    {{ message }}
  </div>
  
  <script>
  var app = new Vue({
    el: '#app',
    data: {
      message: 'Hello Vue!'
    }
  })
  </script>
  ```

  - 之后在页面上会看见`Hello Vue!`，而且**数据是具有响应式的**，也就是说修改`app.message`，页面上的值也会发生变化
  - 下面来分析如何简单实现`Vue`的响应式

## 分析

### 数据渲染

- 先思考怎么把数据渲染到视图上，一般我们的方法都是通过JavaScript操作DOM，例如

  ```javascript
  let msg = '123';
  document.querySelector('#app').innerHTML = msg;
  ```

  效果如下：

  ![](1.png)

- 之后修改变量时，想要修改视图，明显需要继续利用JavaScript操作DOM，**重复的操作明显可以使用函数封装**，可以先把JavaScript修改DOM的操作封装为一个函数，然后在适当地时候调用即可，例如

  ```javascript
  let msg = '123';
  update();
  //异步操作效果更清晰
  setTimeout(() => {
      msg = '456';
      update();
  }, 1000);
  //update函数
  function update() {
      document.querySelector('#app').innerHTML = msg;
  }
  ```

效果如下（1秒后数据变化，DOM也变化）：

![](2.gif)

### 数据劫持

- 那我们想做到变量`msg`变化，视图就变化（Vue所做到的），该怎么办呢？答案很清晰使用某种办法，**在`msg`变化之后立即调用`update`函数，那么这里就要使用数据劫持**

#### getter和setter

- 在JavaScript的属性描述符中有这两种特殊的描述符`setter`和`getter`，它们的作用如下

  ```javascript
  let obj = {
      _a: 1
  }
  Object.defineProperty(obj, 'a', {
      get() {
          return this._a
      },
      set(val) {
          this._a = val
      }
  })
  ```

  效果如下：

  ![](3.gif)

  - 可以看到`setter`和`getter`代理了`obj._a`（**ES6提出了`Proxy`来规范代理同时也是Vue3数据劫持所采用的方案**）

### 实现同步更新

- 利用属性描述符，我们就可以做到数据和视图同步更新

  ```javascript
  //data数据
  let data = {
      _msg: ''
  };
  //数据劫持
  Object.defineProperty(data, 'msg', {
      set(val) {
          this._msg = val;
          //数据更新完成后更新视图
          update();
      },
      get() {
          return this._msg;
      }
  })
  
  function update() {
      document.querySelector('#app').innerHTML = data.msg;
  }
  
  //测试代码
  data.msg = '123';
  setTimeout(() => {
      data.msg = '456'
  }, 1000);
  ```

  效果如下：

  ![](4.gif)

  - 至此最基础简单的响应式就完成了

### 依赖收集

- 由于数据可能会和其他数据产生互动，可能渲染到多个组件中，确保它的正确渲染Vue采用了观察者模式设计了依赖收集机制

- 订阅者（简化版）

  ```javascript
  class Dep {
      constructor() {
          this.subs = [];//订阅数组
      }
      addSub(watcher) {
          this.subs.push(watcher)//添加一个watcher
      }
      notify() {
          this.subs.forEach((watcher) => {
              watcher.update()//执行所有watcher的update
          })
      }
  }
  ```

  - 订阅者收集所有watcher，并在合适的时候通知所有watcher更新视图

- 观察者（简化版）

  ```javascript
  class Watcher {
      constructor(obj, key, cb) {
          Dep.target = this;//确保Dep的对象是这个watcher
          this.val = obj[key];//获取值，主要为了触发getter
          this.obj = obj;
          this.key = key;
          this.cb = cb;//更新逻辑
          Dep.target = null//防止watcher重复
      }
      update() {
          this.val = this.obj[this.key];//获取最新的值
          this.cb(this.val)//根据回调函数的逻辑更新视图
      }
  }
  ```

  - 注意每个watcher的构造函数实例化调用一次，触发一次getter，则每个watcher都可以负责一个值的视图更新逻辑

- 数据劫持（简化版）

  ```javascript
  let dep = new Dep();
  for (const key in data) {//遍历data，劫持每一个属性
      if (Object.hasOwnProperty.call(data, key)) {//确保在data上
          let value = data[key];//获取原值
          Object.defineProperty(data, key, {
              get() {
                  if (Dep.target) {//配合watcher的构造函数
                      dep.addSub(Dep.target)
                  }
                  return value
              },
              set(val) {
                  value = val;
                  dep.notify();//值发生了变化，通知所有watcher进行更新
              }
          })
      }
  }
  //生成观察者
  new Watcher(data, 'pri1', (val) => {
      //pri1的更新逻辑
      document.querySelector('.pri1').innerHTML = 'pri1:' + val;
      document.querySelector('.total').innerHTML = 'total:' + (val + data.pri2);
  })
  new Watcher(data, 'pri2', (val) => {
      //pri2的更新逻辑
      document.querySelector('.pri2').innerHTML = 'pri2:' + val;
      document.querySelector('.total').innerHTML = 'total:' + (val + data.pri1)
  })
  ```

  - 实际Vue当中不会一个一个手动生成watcher，会根据表达式推断视图更新逻辑

  效果如下：

  ![](5.gif)

- 至此Vue响应式最基础的原理就差不多了，但Vue对此做了很多优化以匹配很多机制

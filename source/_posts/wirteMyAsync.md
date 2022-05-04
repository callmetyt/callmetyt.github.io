---
title: 手动实现一个简单的async
date: 2021-5-31 16:30:00
categories:
- javascript
tags:
- javascript
- async
description: async的出现宣告了ECMAscript对Promise+Generator异步编程的标准支持，可它背后的原理是什么呢？
cover: /img/cover_1.webp

---

## 基本概念

- **async函数只是promise+generator函数的语法糖**
  - 具体可见《你不懂的JavaScript》中卷

- promise+generator函数可以使用同步计划书写异步逻辑，更符合人类习惯去书写异步代码
  - **async函数进一步简化了上述代码**

## 手写源码连接

[来自晨曦时梦见兮大佬](https://juejin.cn/post/6844904102053281806)

## 使用示例

```javascript
const getData = () => new Promise(resolve => setTimeout(() => resolve("data"), 1000))

async function test() {
  const data = await getData()
  console.log('data: ', data);
  const data2 = await getData()
  console.log('data2: ', data2);
  return 'success'
}

// 这样的一个函数 应该再1秒后打印data 再过一秒打印data2 最后打印success
test().then(res => console.log(res))
```

代码效果

{% asset_img 1.gif 效果图 %}

- 可以看到使用async函数，await标识出异步代码，js执行时就会在进行异步阻塞，整个async函数会停止下来
  - 关于async函数具体使用见[阮一峰大佬的教学](https://es6.ruanyifeng.com/#docs/async)

## 实现思路

### generator函数形式

```javascript
const getData = () =>
  new Promise((resolve) => {
    setTimeout(() => {
      resolve("haha");
    }, 1000);
  });

function* testG() {
  const data = yield getData();
  console.log("data:", data);
  const data2 = yield getData();
  console.log("data2:", data2);
  return "success";
}
```

- getData函数不变
- **test函数async使用generator函数标识符*代替，所有await使用yield代替**

### 手动实现

```javascript
var gen = testG()//获取迭代器实例

var dataPromise = gen.next()//获取第一个promise

dataPromise.then((value1) => {
    // data1的value被拿到了 继续调用next并且传递给data 并获取第二个promise
    var data2Promise = gen.next(value1)
    
    // console.log('data: ', data);
    // 此时就会打印出data
    
    data2Promise.value.then((value2) => {
        // data2的value拿到了 继续调用next并且传递value2
         gen.next(value2)
         
        // console.log('data2: ', data2);
        // 此时就会打印出data2
    })
})
```

- 手动调用generator函数，实现异步操作

### 利用高阶函数实现自动执行

- 期望的终极形态

  ```javascript
  asyncToGenerator(testG).then((res) => console.log(res));
  //传入一个生成器函数，asyncToGenerator函数会自动执行完，并返回一个promise
  ```

  - 那么asyncToGenerator函数最后应该返回一个promise

- **连续的promise调用**

  ```javascript
  function* testG() {
    const data = yield getData();
    console.log("data:", data);
    const data2 = yield getData();
    console.log("data2:", data2);
    return "success";
  }
  ```

  - generator函数可能存在多个promise，需要一步一步自己调用解包

  - 怎么重复调用是重点，而解决思路是找到链式调用的节点

  ```javascript
  //retPromise就是generator函数返回的promise
  retPromise.then((res)=>{
      //it就是迭代器 即在一个promise状态确定后，根据状态进行下一步。例如resolve下，应该将值返回去，接受下一个promise
      it.next(res);
  })
  ```

  - 将这里的逻辑抽离出来形成函数，然后在合适的时间调用即可

- 最终实现的代码

  ```javascript
  function asyncToGenerator(generatorFunc) {
    //获取generator函数的迭代器
    let it = generatorFunc();
    //返回一个新的promise
    return new Promise((resolve, reject) => {
      //step函数用于调用下一个promise
      //key为迭代器运行方式，val为迭代器要传递进去的参数
      function step(key, val) {
        //generator函数返回的value，由于try属于特殊的上下文，变量需要定义在外部
        let retPromise;
  
        try {
          retPromise = it[key](val); //迭代器运行
        } catch (e) {
          return reject(e); //出错直接由外部捕获
        }
  
        const { value, done } = retPromise; //从generator函数返回的值
        //value即是promise,done用于确定迭代器是否迭代完毕
  
        if (done) {
          //如果迭代完毕了，直接向外部抛出最后的值
          return resolve(value);
        } else {
          //如果没有，调用generator函数返回的promise，注册then事件
          return value.then(
            (val) => {
              //generator函数返回的promise变为resolve时，把获取的值传入step函数，并指示迭代器进行下一步
              step("next", val);
            },
            (err) => {
              //逻辑同上
              step("throw", err);
            }
          );
        }
      }
      //开始迭代，第一个不需要参数，直接下一步即可
      step("next");
    });
  }
  ```

- 最终效果

  {% asset_img 2.gif 效果图 %}

--


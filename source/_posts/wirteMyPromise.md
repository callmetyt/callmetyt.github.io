---
title: 手动实现一个简单的Promise
date: 2021-05-19 12:26:11
tags: 
 - javascript
 - promise
categories:
 - javascript
description: 手动实现ES6的Promise的简化版，重点是理解并实现Promise的链式调用
cover: img/cover_3.webp

---
- 自定义的promise

  ```javascript
  function myPromise(fn) {
      this.state = 'pending'; //记录状态，只有三种可能pending resolved rejected
      this.val = null; //promise携带值
      this.onResolvedCallback = []; //待调用的回调函数，pending状态下调用then时需要 
      this.onRejectedCallback = [];
  
      this.resolve = (val) => {
          if (this.state === 'pending') { //保证状态确定后不会改变
              this.state = 'resolved';
              this.val = val;
              //状态变化后，取出所有待调用的回调函数，进行处理。例如：then的第一个回调函数
              for (let i = 0; i < this.onResolvedCallback.length; i++) {
                  this.onResolvedCallback[i](val);
              }
          }
      }
  
      this.reject = (reason) => {
          if (this.state === 'pending') { //保证状态确定后不会改变
              this.state = 'rejected';
              this.val = reason;
              //和resolve类似
              for (let i = 0; i < this.onRejectedCallback.length; i++) {
                  this.onRejectedCallback[i](reason);
              }
          }
      }
  
      this.then = (resolveFn, rejectFn) => {
          //promise值穿透，并保证默认处理函数
          resolveFn = typeof resolveFn === 'function' ? resolveFn : function (v) { return v };
          rejectFn = typeof rejectFn === 'function' ? rejectFn : function (r) { return r };
  
          if (this.state === 'resolved') { //调用then时状态是resolved
              return new myPromise((resolve, reject) => { //then永远返回一个promise，用于链式调用
                  try { //try捕获错误，用来reject
                      let next = resolveFn(this.val); //调用用户自定义的resolve处理函数，把值传出去，并获取用户返回的值
                      if (next instanceof myPromise) { //resolve处理函数返回的是promise时，调用then解包
                          next.then(resolve, reject)
                          //注意，此处将外部（then函数定义）即将返回的promise的resolve传入了用户自定义的resolve处理函数返回的promise中，当用户的promise状态改变时，就会调用此处的resolve并将值传给外部的promise，从而可以使用最外部的then接收，构成返回 new promise 的promise链式调用
                      }
                      else {
                          resolve(next) //否则直接resolve用户传进来的值
                      }
                  } catch (e) {
                      rejectFn(e)
                  }
              })
          }
  
          if (this.state === 'rejected') { //reject的状态，逻辑和上面差不多
              return new myPromise((resolve, reject) => {
                  try {
                      let next = rejectFn(this.val);
                      if (next instanceof myPromise) {
                          next.then(resolve, reject)
                      }
                  } catch (e) {
                      rejectFn(e)
                  }
              })
          }
  
          if (this.state === 'pending') { //调用then时，状态未确定
              return new myPromise((resolve, reject) => {
                  //由于状态未确定，先把状态确定时要执行的逻辑保存起来，等到状态确定时，执行
                  this.onResolvedCallback.push(function (val) { //resolved逻辑保存
                      try {
                          let next = resolveFn(val);
                          if (next instanceof myPromise) {
                              next.then(resolve, reject)
                          };
                          resolve(next);
                      } catch (e) {
                          reject(e)
                      }
                  })
  
                  this.onRejectedCallback.push(function (reason) { //reject逻辑保存
                      try {
                          let next = rejectFn(reason);
                          if (next instanceof myPromise) {
                              next.then(resolve, reject)
                          }
                          reject(next)
                      } catch (e) {
                          reject(e)
                      }
                  })
              })
          }
      }
  
      try { //尝试执行promise构造函数的代码，并将自身定义好的resolve,reject传入
          fn(this.resolve, this.reject);
      } catch (e) {
          this.reject(e)
      }
  }
  ```
  
  测试样例
  
  ```javascript
  new myPromise((resolve, reject) => {
      resolve(1)
  }).then((val) => {
      console.log(val);
      return new myPromise((resolve) => {
          setTimeout(() => {
              resolve(2)
          }, 1000);
      })
  }).then((val) => {
      console.log(val);
  })
  ```
  
  + 异步throw无法捕获
  + 通过this定义函数，导致每个promise实例不共用方法，性能影响
  
- all,race手写

  ```javascript
  Promise.myAll = (promiseArr) => {
      return new Promise((resolve, reject) => {
          let count = 0, all = promiseArr.length;
          let resArr = [];
          for (let i = 0; i < all; i++) {
              promiseArr[i].then((val) => {
                  count++;
                  resArr[i] = val;
                  if (count === all) { //主要通过promise数量确定是否结束
                      resolve(resArr)
                  }
              }, (err) => {
                  reject(err)
              })
          }
      })
  }
  ```

  ```javascript
  //race更简单 待会写....
  ```

  

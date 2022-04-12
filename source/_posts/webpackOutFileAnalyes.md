---
title: 一次对webpack输出文件的分析
date: 2021/5/25 17:24:34
tags:  
- webpack
categories:
- javascript
description: webpack（v5.37.0）输出文件分析,了解webpack打包原理
cover: /img/cover_4.webp

---
## webpack 5.37.0 的输出文件分析

- 入口文件

  ```javascript
  //main.js
  import show from "./tmp2";
  
  show(123);
  console.log("1111");
  ```

- 依赖文件

  ```javascript
  //tmp2.js
  export default function show(arg) {
  	console.log("show me!");
  	console.log(`arg is ${arg}`);
  }
  ```

- webpack.config.js

  ```javascript
  module.exports={
      //其余配置省略
      devtool:"sourcce-map"
  }
  ```

- 输出文件

  ```javascript
  /******/ (() => { // webpackBootstrap
  /******/ 	"use strict";
  /******/ 	var __webpack_modules__ = ({
  
  /***/ "./mySrc/js/tmp2.js":
  /*!**************************!*\
    !*** ./mySrc/js/tmp2.js ***!
    \**************************/
  /***/ ((__unused_webpack_module, __webpack_exports__, __webpack_require__) => {
  
  __webpack_require__.r(__webpack_exports__);
  /* harmony export */ __webpack_require__.d(__webpack_exports__, {
  /* harmony export */   "default": () => (/* binding */ show)
  /* harmony export */ });
  function show(arg) {
      console.log("show me!");
      console.log(`arg is ${arg}`);
  }
  
  
  /***/ })
  
  /******/ 	});
  /************************************************************************/
  /******/ 	// The module cache
  /******/ 	var __webpack_module_cache__ = {};
  /******/ 	
  /******/ 	// The require function
  /******/ 	function __webpack_require__(moduleId) {
  /******/ 		// Check if module is in cache
  /******/ 		var cachedModule = __webpack_module_cache__[moduleId];
  /******/ 		if (cachedModule !== undefined) {
  /******/ 			return cachedModule.exports;
  /******/ 		}
  /******/ 		// Create a new module (and put it into the cache)
  /******/ 		var module = __webpack_module_cache__[moduleId] = {
  /******/ 			// no module.id needed
  /******/ 			// no module.loaded needed
  /******/ 			exports: {}
  /******/ 		};
  /******/ 	
  /******/ 		// Execute the module function
  /******/ 		__webpack_modules__[moduleId](module, module.exports, __webpack_require__);
  /******/ 	
  /******/ 		// Return the exports of the module
  /******/ 		return module.exports;
  /******/ 	}
  /******/ 	
  /************************************************************************/
  /******/ 	/* webpack/runtime/define property getters */
  /******/ 	(() => {
  /******/ 		// define getter functions for harmony exports
  /******/ 		__webpack_require__.d = (exports, definition) => {
  /******/ 			for(var key in definition) {
  /******/ 				if(__webpack_require__.o(definition, key) && !__webpack_require__.o(exports, key)) {
  /******/ 					Object.defineProperty(exports, key, { enumerable: true, get: definition[key] });
  /******/ 				}
  /******/ 			}
  /******/ 		};
  /******/ 	})();
  /******/ 	
  /******/ 	/* webpack/runtime/hasOwnProperty shorthand */
  /******/ 	(() => {
  /******/ 		__webpack_require__.o = (obj, prop) => (Object.prototype.hasOwnProperty.call(obj, prop))
  /******/ 	})();
  /******/ 	
  /******/ 	/* webpack/runtime/make namespace object */
  /******/ 	(() => {
  /******/ 		// define __esModule on exports
  /******/ 		__webpack_require__.r = (exports) => {
  /******/ 			if(typeof Symbol !== 'undefined' && Symbol.toStringTag) {
  /******/ 				Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' });
  /******/ 			}
  /******/ 			Object.defineProperty(exports, '__esModule', { value: true });
  /******/ 		};
  /******/ 	})();
  /******/ 	
  /************************************************************************/
  var __webpack_exports__ = {};
  // This entry need to be wrapped in an IIFE because it need to be isolated against other modules in the chunk.
  (() => {
  /*!**************************!*\
    !*** ./mySrc/js/main.js ***!
    \**************************/
  __webpack_require__.r(__webpack_exports__);
  /* harmony import */ var _tmp2__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(/*! ./tmp2 */ "./mySrc/js/tmp2.js");
  
  (0,_tmp2__WEBPACK_IMPORTED_MODULE_0__.default)(123);
  console.log("1111");
  
  })();
  
  /******/ })()
  ;
  //# sourceMappingURL=build.js.map
  ```
  
  - 看不懂不要紧，看完下面的分块分析，再看看整体代码

### 关键分析

- 整体是一个立即执行函数（防止变量污染作用域）
- 内部先分析定义的一些函数
- 再分析模块数组
- 最后分析入口模块的执行

### __webpack_require\_\_函数

```javascript
/******/ 	// The module cache
/******/ 	var __webpack_module_cache__ = {};
/******/ 	
/******/ 	// The require function
/******/ 	function __webpack_require__(moduleId) {
/******/ 		// Check if module is in cache
/******/ 		var cachedModule = __webpack_module_cache__[moduleId];
/******/ 		if (cachedModule !== undefined) {
/******/ 			return cachedModule.exports;
/******/ 		}
/******/ 		// Create a new module (and put it into the cache)
/******/ 		var module = __webpack_module_cache__[moduleId] = {
/******/ 			// no module.id needed
/******/ 			// no module.loaded needed
/******/ 			exports: {}
/******/ 		};
/******/ 	
/******/ 		// Execute the module function
/******/ 		__webpack_modules__[moduleId](module, module.exports, __webpack_require__);
/******/ 	
/******/ 		// Return the exports of the module
/******/ 		return module.exports;
/******/ 	}
```

- 缓存优化

  ```javascript
  var __webpack_module_cache__ = {}; //缓存数组
  //...
  var cachedModule = __webpack_module_cache__[moduleId]; //从缓存中找出模块
  if (cachedModule !== undefined) {
      return cachedModule.exports;//如果找到了就直接返回
  }
  ```

  - 逻辑很简单的优化机制

- require 模仿

  ```javascript
  function __webpack_require__(moduleId) {
      // ...
      // 创建一个模块
      var module = (__webpack_module_cache__[moduleId] = {
          //同时创建了缓存
          exports: {},
      });
  
      // 执行模块代码
      __webpack_modules__[moduleId](module, module.exports, __webpack_require__);//模块数组中每一个模块都是一个函数，下面会看到模块内部怎么对传入的module进行组装
  
      // 返回模块抛出的内容
      return module.exports;
  }
  ```

  - 用于在浏览器中模仿commonjs或者ES6 module

### \__webpack_require\_\_工具函数

- hasOwnProperty 封装 \__webpack_require\_\_.o

  ```javascript
  (() => {
      __webpack_require__.o = (obj, prop) =>
      Object.prototype.hasOwnProperty.call(obj, prop);
  })();
  ```

  - 只是个简单的封装，功能基本不变

- **定义函数 \__webpack_require\_\_.d**

  - 核心功能函数，成功模仿Commonjs规范，抛出了模块

  ```javascript
  (() => {
      // 即将抛出的exports上定义代码
      __webpack_require__.d = (exports, definition) => {//传入exports和定义
          for (var key in definition) {
              if (
                  __webpack_require__.o(definition, key) &&
                  !__webpack_require__.o(exports, key)
              ) //上面提到的封装函数，判定属性是否存在以及是否发生覆盖
              {
                  Object.defineProperty(exports, key, {
                      enumerable: true,
                      get: definition[key],
                  });//exports上定义getter，也就是module.exports访问时，利用get定位到模块中抛出的同名属性
              }
          }
      };
  })();
  ```

  - 结合下面的模块解析会更清晰

- 标记函数 \__webpack_require\_\_.r

  ```javascript
  (() => {
      // 标记这是个模块
      __webpack_require__.r = (exports) => {
          if (typeof Symbol !== "undefined" && Symbol.toStringTag) {//Symbol.toStringTag可以修改typeof的返回值
              Object.defineProperty(exports, Symbol.toStringTag, { value: "Module" });
          }
          Object.defineProperty(exports, "__esModule", { value: true });//不支持Symbol的替代品
      };
  })();
  ```

### 模块数组

- 此处只有一个依赖模块（main.js作为入口模块，webpack5不归类为模块数组）

  ```javascript
  var __webpack_modules__ = {
      /***/ "./mySrc/js/tmp2.js"://本身是个键值对,键就是引用路径,值是一个函数
      /*!**************************!*\
    !*** ./mySrc/js/tmp2.js ***!
    \**************************/
      /***/ (
          __unused_webpack_module,
          __webpack_exports__,
          __webpack_require__
      ) => {
          __webpack_require__.r(__webpack_exports__);//首先标记模块
          /* harmony export */ __webpack_require__.d(__webpack_exports__, {
              /* harmony export */ default: () => /* binding */ show,
              /* harmony export */
          });//这一步利用定义函数，将本模块抛出的属性定义在module.exports上，从而使外部能够正常访问
          //下面就是源代码
          function show(arg) {
              console.log("show me!");
              console.log(`arg is ${arg}`);
          }
  
          /***/
      },
  };
  ```

### 入口模块

```javascript
(() => {
    /*!**************************!*\
  !*** ./mySrc/js/main.js ***!
  \**************************/
    /* harmony import */ var _tmp2__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(
        /*! ./tmp2 */ "./mySrc/js/tmp2.js"
    );//利用__webpack_require__函数模仿require读取了tmp2模块

    (0, _tmp2__WEBPACK_IMPORTED_MODULE_0__.default)(123);//调用tmp2抛出的show函数
    console.log("1111");//入口源代码
})();
```

- 关键是 \__webpack_require\_\_ 函数成功返回了模块抛出的属性，这一点必须理解，之后的内容就很简单了





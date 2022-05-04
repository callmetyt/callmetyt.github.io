---
title: ReactSourceDebug
date: 2022-03-21 07:52:01
tags:
 - React
 - Javascript
 - source
categories:
 - React
description: React的一次源码debug学习
top_image: https://pic2.zhimg.com/v2-8c3223d9228013e069df6a302e81c38f_720w.png?source=d16d100
cover: /img/cover_5.webp

---
## 前言

- React源码版本：`17.0.2`
- Debug Demo 如下：

```typescript
class Test extends React.Component<any, { text: string }> {
  constructor(props: any) {
    super(props);
    this.state = {
      text: "hello",
    };
  }
  render() {
    return (
      <div>
        <span>{this.state.text}</span>
        <span>normal</span>
        <button
          onClick={() => {
            debugger;
            this.setState({ text: "check" });
          }}
        >
          setState
        </button>
      </div>
    );
  }
}

ReactDOM.render(<Test />, document.getElementById("root"));
```

- 根据DEMO，本次学习仅针对React在Web层的渲染
  - 页面的初始渲染
  - 点击按钮后的一次更新渲染
- 辅助学习文章——[Lam大佬的React源码解析文章](https://www.zhihu.com/people/lam-14-21-74)
- **本文仅是学习记录，记录一些源码的理解，并非详细分析源码**

## 初次渲染

### 入口

```typescript
ReactDOM.render(<Test />, document.getElementById("root"));
```

- 通过传入入口组件（通常是`<App />`）进行整体页面的渲染

### JSX处理

- 注意下`webpack`对所有的JSX进行了处理，比如入口代打包成了这样

```typescript
react_dom__WEBPACK_IMPORTED_MODULE_1__.render( /*#__PURE__*/(0,react_jsx_dev_runtime__WEBPACK_IMPORTED_MODULE_4__.jsxDEV)(Test, {}, void 0, false, {
  fileName: _jsxFileName,
  lineNumber: 52,
  columnNumber: 17
}, undefined), document.getElementById("root"));
```

- 可以看到所有的JSX都包裹在了一个函数中，函数中有两个关键动作
  - 对JSX写法进行验证，是否符合语法
  - 对JSX进行解析

```typescript
// DEMO中的Test解析返回的结果
{
  $$typeof: Symbol(react.element),
  type: class Test /* 这里省略Test的结构 */,
  key: null,
  ref: null,
  props: {
  },
  _owner: null,
  _store: {
  },
}
```

### render

- 首次渲染时会去创建`root`，同时代码首次涉及到React16加入的`Fiber`数据结构

```typescript
// 创建FiberRoot的函数
function createFiberRoot(containerInfo, tag, hydrate, hydrationCallbacks) {
  var root = new FiberRootNode(containerInfo, tag, hydrate);
  var uninitializedFiber = createHostRootFiber(tag);
  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;
  initializeUpdateQueue(uninitializedFiber);
  return root;
}
```

附上此时的调用栈

{% asset_img 3.png 调用栈 %}

#### Fiber

- 源码整体Debug下来，每一个Fiber对应`JSX`的每一个节点（不同于真实DOM树），Fiber树包括了`<App />`这样的组件
- React整体基于Fiber树进行更新，可以说Fiber树就是React的虚拟DOM
- Fiber树的结构如下，以DEMO为例：

{% asset_img 1.png Fiber树结构 %}

- **每个Fiber只有一个父节点、一个子节点、一个兄弟节点**，可能没有父节点（root），可能没有子节点（两个span和button）、可能没有兄弟节点，但有也不会超过一个

  - 所以Fiber有以下的数据结构：

  ```typescript
  interface Fiber {
      return:FiberNode, //父节点
      sibiling:FiberNode, //兄弟节点
      child:FiberNode, //子节点
  }
  ```

- Fiber节点上记录了对应节点的所有需要渲染的属性，例如：rootFiber有下面这些属性（**某些属性是rootFiber独有的**）：

{% asset_img 2.png Fiber属性 %}

#### 事件监听

- React会在root节点上监听所有可能的事件

```typescript
// 创建rootFiber之后，调用的监听函数
function listenToAllSupportedEvents(rootContainerElement) {
  {
    if (rootContainerElement[listeningMarker]) {
      return;
    }
    rootContainerElement[listeningMarker] = true;
    allNativeEvents.forEach(function (domEventName) {
      if (!nonDelegatedEvents.has(domEventName)) {
        listenToNativeEvent(domEventName, false, rootContainerElement, null);
      }

      listenToNativeEvent(domEventName, true, rootContainerElement, null);
    });
  }
}
```

- `allNativeEvents`包含了React预定的所有事件，React将事件分成了三组，分为三种优先级，例如优先级最低的一组为：

```typescript
var discreteEventPairsForSimpleEventPlugin = ['cancel', 'cancel', 'click', 'click', 'close', 'close',/*...其余好多事件*/];
```

- 在`root`上监听事件之后，所有事件都会在`root`上触发对应处理函数，并后续调用`dispatchEvent`函数，**这个函数会统一调度和处理所有事件，并在最后进行更新**

### updateContainer

- 更新（处理Fiber）函数
  - `unbatchedUpdates`涉及到了React的合成事件系统（没有详细了解...）

```typescript
unbatchedUpdates(function () {
    updateContainer(children, fiberRoot, parentComponent, callback);
});
```

- 函数内部对React的更新算法进行了初始化
  - `updatequeue`
  - `lane`算法

### workLoopSync

- 此函数对Fiber树进行遍历，同时**创建和处理**所有的Fiber节点

```typescript
function workLoopSync() {
    // Already timed out, so perform work without checking if we need to yield.
    while (workInProgress !== null) {
        performUnitOfWork(workInProgress);
    }
}
```

此时的调用栈：

{% asset_img 4.png 调用栈 %}

### Fiber遍历

- `workLoopSync`对**Fiber树的遍历（此时其实还只有一个rootFiber根）**主要调用了两个函数
  - `beginWork`
  - `completeUnitOfWork`

- 这两个函数都会根据Fiber节点的不同，进行不同的处理
  - **`beginWork`函数会对每一种Fiber节点调用`reconcileChildren`函数，通过传入Fiber的child（这里的child是通过JSX解析获取的，还不是Fiber）创建对应的Fiber并且返回这个Fiber**
  - **`completeUnitOfWork`函数则会根据Fiber的类型真正创建DOM节点，并对其初始化**
- 同时在`workLoopSync`内部会按照下面的顺序进行遍历（**b代表`beginWork`，b1代表第一步调用`beginWork`处理对应节点**）：{% asset_img 5.png 更新顺序 %}

- 可以看出，Fiber树遍历之后，`workLoopSync`会返回Fiber树的根节点，也就是`root`节点

### commitRoot

- `commitRoot`函数则会真正将Fiber树对应的真实DOM结构渲染到页面上

```typescript
// commitRoot函数内部调用渲染真实DOM的部分代码
do {
    {
        invokeGuardedCallback(null, commitMutationEffects, null, root, renderPriorityLevel);
        if (hasCaughtError()) {
            if (!(nextEffect !== null)) {
                {
                    throw Error( "Should be working on an effect." );
                }
            }
            var _error = clearCaughtError();
            captureCommitPhaseError(nextEffect, _error);
            nextEffect = nextEffect.nextEffect;
        }
    }
} while (nextEffect !== null);
```

- 代码中还执行了另外两次相似函数，分别在上述代码前后执行
  - `invokeGuardedCallback(null, commitBeforeMutationEffects, null);`
  - `invokeGuardedCallback(null, commitLayoutEffects, null, root, lanes);`

此时的调用栈：{% asset_img 6.png 调用栈 %}

- 在上述代码执行过后，页面上才会真正渲染出真实DOM


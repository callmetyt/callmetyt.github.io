---
title: ReactSourceHookDebug
date: 2022-03-27 10:51:50
tags:
 - React
 - source
 - javascript
categories:
 - React
description: React源码Debug学习体会——Hook篇
top_image: https://pic2.zhimg.com/v2-8c3223d9228013e069df6a302e81c38f_720w.png?source=d16d100
cover: /img/cover_3.webp

---
## 前言

- React源码版本：`17.0.2`
- Debug Demo 如下：

```typescript
const Test = () => {
  const [num, setNum] = useState(0);
  useEffect(() => {
    console.log("effect!");
  });
  return (
    <button
      onClick={() => {
        setNum((prev) => {
          return prev + 1;
        });
      }}
    >
      {num}
    </button>
  );
};
```

- **本文章记录学习React Hook源码时的一些收获**

## Hook流程

### beginWork

- 跳过`rootContainer`的创建，lanes算法初始化等等代码，首先关注`<Test />`组件的Fiber初始化，根据DEMO，**此时`beginWork`已经处理rootFiber并创建了`<Test />`组件的Fiber**，现在要对其进行初始化

此时`<Test />`的Fiber（已省略部分属性）

```typescript
{
  tag: 2,
  key: null,
  /*可以看到webpack处理后的Test函数中的JSX都被jsxDev解析函数包裹*/
  elementType: () => {
    _s();
    const [num, setNum] = (0,react__WEBPACK_IMPORTED_MODULE_0__.useState)(0);
    (0,react__WEBPACK_IMPORTED_MODULE_0__.useEffect)(() => {
      debugger;
      console.log("effect!");
    });
    return /*#__PURE__*/(0,react_jsx_dev_runtime__WEBPACK_IMPORTED_MODULE_4__.jsxDEV)("button", {
      onClick: () => {
        debugger;
        setNum(prev => {
          return prev + 1;
        });
      },
      children: num
    }, void 0, false, {
      fileName: _jsxFileName,
      lineNumber: 59,
      columnNumber: 5
    }, undefined);
  },
  type: () => {/*webpack处理后的Test函数和elementType一致*/},
  stateNode: null,
  return: {/*rootFiber*/},
  child: null,
  sibling: null,
  index: 0,
  ref: null,
  pendingProps: {},
  memoizedProps: null,
  updateQueue: null,
  memoizedState: null,
  dependencies: null,
  mode: 8,
  flags: 2,
  nextEffect: null,
  firstEffect: null,
  lastEffect: null,
  alternate: null,
}
```

此时的调用栈

![1](11.png)

- Fiber的tag为2，由如下源码可知，是因为React还不知道是`class`还是`function`

> var IndeterminateComponent = 2; *// Before we know whether it is function or class*

### renderWithHooks

- 后续调用函数去专门处理tag为2的组件，以下是React判断组件时`class`还是`function`的代码

```typescript
if (Component.prototype && typeof Component.prototype.render === 'function'){
    //是class组件，...
}
```

- 然后保存Fiber进行处理

```typescript
ReactCurrentOwner$1.current = workInProgress; // workInProgress 就是当前处理的Fiber
value = renderWithHooks(null, workInProgress, Component, props, context, renderLanes);
```

- **之后会调用这个代码，为当前环境设置hook，之后会使用`throwError`版本覆盖，这也是为什么hook只能在`function`组件中使用的原因**

```typescript
ReactCurrentDispatcher$1.current = HooksDispatcherOnMountInDEV;
// 而HooksDispatchOnMountInDEV则保存了所有React的hook（当然也有update版本）
```

下图为部分hook

![22](22.png)

- 之后调用`function`组件，执行用户代码，获取`child`

调用栈如图

![33](33.png)

### useState

```typescript
// useState源码 省略部分源码
useState: function (initialState) {
    var prevDispatcher = ReactCurrentDispatcher$1.current; // 这里就会获取正常运行的Hook代码
    ReactCurrentDispatcher$1.current = InvalidNestedHooksDispatcherOnMountInDEV;

    try {
        return mountState(initialState); // useState主要执行的逻辑
    } finally {
        ReactCurrentDispatcher$1.current = prevDispatcher;
    }
}
```

- 接受用户的值作为`initialState`，并最后调用`mountState`

```typescript
//mountState 源码
function mountState(initialState) {
    var hook = mountWorkInProgressHook(); // 生成hook对象

    if (typeof initialState === 'function') { // 如果initialState是函数，就会执行并获取返回值
        // $FlowFixMe: Flow doesn't like mixed types
        initialState = initialState();
    }

    hook.memoizedState = hook.baseState = initialState; // 获取初始值，并保存（闭包）
    var queue = hook.queue = {
        pending: null,
        dispatch: null,
        lastRenderedReducer: basicStateReducer, // 更新函数
        lastRenderedState: initialState
    };
    var dispatch = queue.dispatch = dispatchAction.bind(null, currentlyRenderingFiber$1, queue); // useState返回的更新函数，已经硬绑定了2个参数
    return [hook.memoizedState, dispatch]; // 返回形式，也是为什么使用 const [val,setVal] = useState(0); 的原因
}
```

- 首先初始化hook对象，用来保存`state`数据、调用队列等

- **`dispatch`函数绑定了当前Fiber，并且调用`lastRenderedReducer`（也就是`basicStateReducer`），最后进行`scheduleUpdateOnFiber`（发起调度更新）**

```typescript
// 已省略大部分代码
function dispatchAction(fiber, queue, action) {
    try {
        var currentState = queue.lastRenderedState; // 获取保存的state
        var eagerState = lastRenderedReducer(currentState, action); // 获取新的state
		// 准备update
        update.eagerReducer = lastRenderedReducer;
        update.eagerState = eagerState;
    } catch (error) {// Suppress the error. It will throw again in the render phase.
    } finally {
        {
            ReactCurrentDispatcher$1.current = prevDispatcher;
        }
    }
	// 执行调度更新
    scheduleUpdateOnFiber(fiber, lane, eventTime);
}
```

- **`action`是用户传入的函数，以下是`lastRenderReducer`源码，所以函数内部可以获取之前的state**

```typescript
function basicStateReducer(state, action) {
  // $FlowFixMe: Flow doesn't like mixed types
  // 如果是函数，就传入之前state并调用，否则直接使用新的state
  return typeof action === 'function' ? action(state) : action;
}
```

### useEffect

```typescript
// useEffect源码
useEffect: function (create, deps) {
    currentHookNameInDev = 'useEffect';
    mountHookTypesDev(); // 添加 hookName 到 hookTpyeDev
    checkDepsAreArrayDev(deps); // 检查useEffect的deps格式是否为数组
    return mountEffect(create, deps); // useEffect的主要逻辑
},
```

- `mountEffect`检查一下`jest`会调用下面这个函数，执行逻辑

```typescript
function mountEffectImpl(fiberFlags, hookFlags, create, deps) {
    // create就是用户订阅的effect函数
    var hook = mountWorkInProgressHook(); // 生成hook对象
    var nextDeps = deps === undefined ? null : deps;
    currentlyRenderingFiber$1.flags |= fiberFlags;
    hook.memoizedState = pushEffect(HasEffect | hookFlags, create, undefined, nextDeps);
}
```

- `hook`的`memoizedState`不同于`useState`，`useEffect`会记录当前的`create`函数、`deps`依赖数据和调用顺序

### commitRoot

- `commitRoot`在执行挂载真正DOM树的操作之前还会执行一个额外的程序

> invokeGuardedCallback(null, commitBeforeMutationEffects, null);

- **`invokeGuardedCallback`是React中用于`DEV`环境下触发事件的，附带有各种错误捕获和消息提示**

- 在经过一系列我看不懂的调用之后，会执行`flushPassiveEffects`函数，在这个函数内部会执行用户定义的`effect函数`

此时的调用栈

![44](44.png)

### commitBeforeMutationEffects

- 经过测试后，debug操作不同，下面`flushPassiveEffects`函数调用时机不同，这里看得很迷（应该是不懂React详细更新算法和逻辑的问题）
  - 单步进入，`flushPassiveEffects`函数会在`commitBeforeMutationEffects`函数中直接执行
  - 单步跳过`commitBeforeMutationEffects`，会在React渲染完成之后单独执行

#### flushPassiveEffects

- 省略了其他代码（例如：这里还处理了`unMountEffects`，只不过没有执行）

```typescript
var mountEffects = pendingPassiveHookEffectsMount; // 这里是收集到的effects，并且实在mount阶段执行的effects
pendingPassiveHookEffectsMount = [];
for (var _i = 0; _i < mountEffects.length; _i += 2) {
    // effects数组每两个一组，前一个是effects，后一个是当前effects对应的fiber
    var _effect2 = mountEffects[_i];
    var _fiber = mountEffects[_i + 1];
    {
        setCurrentFiber(_fiber); // 设置当前环境fiber
        {
            // 使用invokePassiveEffectCreate执行effects函数
            invokeGuardedCallback(null, invokePassiveEffectCreate, null, _effect2);
        }
        if (hasCaughtError()) {
            // ...
        }
        resetCurrentFiber();
    }
}
```

- 函数最后还会调用下面这个函数，这样会去清除`syncQueue`队列内部的更新事件，这也是为什么如果在`useEffect`中设置`state`会导致无限更新的原因

```typescript
flushSyncCallbackQueue(); 
// 这个函数内部会调用flushSyncCallbackQueue()
```

#### invokePassiveEffectCreate

- 真正执行`effects`函数的代码

```typescript
function invokePassiveEffectCreate(effect) {
    var create = effect.create;
    effect.destroy = create();
}
// effects如下
{
  tag: 5, // 标识hook类型
  create: () => { // 这个就是用户的effect函数
    debugger;
    console.log("effect!");
  },
  destroy: undefined, // 存储可能存在的销毁函数
  deps: null, // 依赖数组
  next: [Circular], // 下一个effect
}
```

### commitMutationEffects

- **之前说过，React在这个函数中把Fiber树对应渲染好的真实DOM树添加到页面上，除此之外还会执行一次特定`effects`**

#### commitHookEffectListUnmount

- **执行所有`effects`的销毁函数，因为组件进行了一次更新**

```typescript
// 部分源码
    do {
      if ((effect.tag & tag) === tag) {
        // Unmount
        var destroy = effect.destroy; // effect 对象
        effect.destroy = undefined;
        if (destroy !== undefined) {
          destroy(); // 这个就是销毁函数
        }
      }
      effect = effect.next; // 执行下一个effect
    } while (effect !== firstEffect);
```

### commitLayoutEffects

- 在真实DOM渲染完成之后类似方式调用的函数

> invokeGuardedCallback(null, commitLayoutEffects, null, root, lanes);

#### commitLifeCycles

- 如同函数名称一样，会处理各种生命周期函数（包括：effects）

- `commitHookEffectListMount`函数在满足某种条件下（没看懂...），执行`effect`函数

```typescript
// 部分源码
if ((effect.tag & tag) === tag) {
    // Mount
    var create = effect.create;
    effect.destroy = create();
    {
        var destroy = effect.destroy;
        if (destroy !== undefined && typeof destroy !== 'function') {
            // ... 很多对返回的destroy函数使用不当的warn
        }
    }
}
```

#### schedulePassiveEffects

- `schedulePassiveEffects`函数会将需要执行的`effect`保存下来，在后续调用？

```typescript
// 部分源码
do {
    var _effect = effect,
        next = _effect.next,
        tag = _effect.tag;
    if ((tag & Passive$1) !== NoFlags$1 && (tag & HasEffect) !== NoFlags$1) {
        // 下面两个函数分别将fiber和hook存入pendingPassiveHookEffectsUnmount和pendingPassiveHookEffectsMount
        // 而pendingPassiveHookEffectsMount就是在flushPassiveEffects函数中获取effect的数组
        enqueuePendingPassiveHookEffectUnmount(finishedWork, effect);
        enqueuePendingPassiveHookEffectMount(finishedWork, effect);
    }
    effect = next;
} while (effect !== firstEffect);
```



---
title: ReactSourceDiffDebug
date: 2022-04-06 07:28:32
tags:
- React
- Diff
categories:
- React
description: React Diff 算法的一次源码Debug学习
top_img: https://pic2.zhimg.com/v2-8c3223d9228013e069df6a302e81c38f_720w.png?source=d16d100b
cover: /img/cover_1.webp

---
## 前言

- React源码版本：`17.0.2`

- DEMO如下

```tsx
const Test = () => {
  debugger;
  const [list, setList] = useState<ListItemType[]>([
    { text: "22 a", key: "a" },
    { text: "13 b", key: "b" },
    { text: "21 c", key: "c" },
    { text: "12 d", key: "d" },
    { text: "32 e", key: "e" },
    { text: "44 f", key: "f" },
  ]);

  return (
    <>
      <ul>
        {list.map((item) => {
          return item ? <li key={item.key}>{item.text}</li> : null;
        })}
      </ul>
      <button
        onClick={() => {
          debugger;
          setList((prev) => {
            let newArr: ListItemType[] = [];
            // 点击按钮之后，随机删除一个li节点，随机挑选一个节点放到第一行
            prev.splice(Math.floor(Math.random() * 5), 1);
            newArr.push(...prev.splice(Math.floor(Math.random() * 4), 1));
            newArr = newArr.concat(prev);
            return newArr;
          });
        }}
      >
        random all
      </button>
    </>
  );
};
```

## Diff解析

### Debug时机

- **首次渲染之后**，点击`random all`按钮随机处理`state`之后执行`performSyncWorkOnRoot`函数时进行Debug
- **`Test`组件根据`hook`获取最新的`state`，后续先对Fiber更新**

### reconcileChildren

- `beginWork`对`ul`标签进行处理时，**会对其`children`进行一次Diff操作**

```typescript
function reconcileChildren(current, workInProgress, nextChildren, renderLanes) {
    // current是当前已经渲染过的Fiber（workInProcess.alternate）
    if (current === null) {
        // 这个是初始化时进行的操作
        workInProgress.child = mountChildFibers(workInProgress, null, nextChildren, renderLanes);
    } else {
        // 这个是Fiber已经存在时的处理（后续更新）
        workInProgress.child = reconcileChildFibers(workInProgress, current.child, nextChildren, renderLanes);
    }
}
```

- `reconcileChildFibers`针对不同的`child`调用不同的函数去处理`Fiber`，**这里我们主要看 React 对多个同级DOM的删除、更新怎么处理的（DOM Diff）**

```typescript
function reconcileChildFibers(returnFiber, currentFirstChild, newChild, lanes) {
    var isObject = typeof newChild === 'object' && newChild !== null;
    if (isObject) {
        switch (newChild.$$typeof) {
            // 这里处理单个child
            case REACT_ELEMENT_TYPE:
                return placeSingleChild(reconcileSingleElement(returnFiber, currentFirstChild, newChild, lanes));
		   // ......... 例如：Fragment
        }
    }
	// child是字符串或者数字的处理
    if (typeof newChild === 'string' || typeof newChild === 'number') {
        return placeSingleChild(reconcileSingleTextNode(returnFiber, currentFirstChild, '' + newChild, lanes));
    }
	// DOM Diff
    if (isArray$1(newChild)) {
        return reconcileChildrenArray(returnFiber, currentFirstChild, newChild, lanes);
    }
	// ...........其他类型
    return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

### reconcileChildrenArray

- **`reconcileChildrenArray`对`childs`进行处理（创建或者复用DOM）**
- 算法浅析如下：

```typescript
function reconcileChildrenArray(returnFiber, currentFirstChild, newChildren, lanes) {
	// 首先对每个child的key进行验证，是否重复等
    // 代码省略

    var resultingFirstChild = null; // 最后返回的Fiber，用于构建Fiber树
    var previousNewFiber = null; // 保存遍历过程中的上一个newFiber
    var oldFiber = currentFirstChild; // 第一个oldFiber
    // React只允许DOM节点复用时向右移动
    // 只要oldFiber的下标比lastPlacedIndex大，就说明这个节点需要右移
    // 具体算法看 placeChild 这个函数
    var lastPlacedIndex = 0; // 标记newFiber中最新已处理节点的下标
    var newIdx = 0; // 新Child的索引
    var nextOldFiber = null; // 下一个oldFiber
	// 第一次遍历，oldChild和newChild同步遍历
    // 比如：oldChild[0]和newChild[0]尝试是否可以复用
    for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
        if (oldFiber.index > newIdx) { // 旧节点在新节点右边，不了解这种情况什么时候发生...
            nextOldFiber = oldFiber;
            oldFiber = null;
        } else {
		   // 获取下个旧节点，比如oldChild[0]的sibling就是oldChild[1]，Fiber树的原因
            nextOldFiber = oldFiber.sibling;
        }
	    // 尝试复用旧Fiber
        // updateSlot内部通过检测key值是否相同来判断是否能够复用
        // 如果能够复用，oldFiber保留，并根据newChild更新下属性即可
        // 同时还有处理别的newFiber的代码，比如newFiber是个textNode，或者newFiber是个Fragment
        var newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx], lanes);
	    //如果不能复用，newFiber为null，此时直接跳出第一次循环
        if (newFiber === null) {
            if (oldFiber === null) {
                oldFiber = nextOldFiber;
            }
            break;
        }
	    // 需要删除这个Fiber
        if (shouldTrackSideEffects) { // shouldTrackSideEffetcs在更新操作中始终为true
            if (oldFiber && newFiber.alternate === null) {
                deleteChild(returnFiber, oldFiber);
            }
        }
	    //更新lastPlacedIndex
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);

        // 获取第一个newFiber用于构建Fiber树，并将上一个newFiber的sibling指向这个newFiber
        if (previousNewFiber === null) { // 第一次循环才会进入这个
            resultingFirstChild = newFiber;
        } else {
            previousNewFiber.sibling = newFiber;
        }
	   // 进入下一次循环，更新标识变量
        previousNewFiber = newFiber;
        oldFiber = nextOldFiber;
    }

    if (newIdx === newChildren.length) {
        //.... 新Child遍历完毕，说明剩下的oldFiber在新DOM树中不存在，直接删除（这里只是标记，DOM操作都在 commitRoot 函数里）
    }

    if (oldFiber === null) {
        //.... 旧Child遍历完毕，说明剩下的newFiber是新添加的，没有可复用Fiber，直接创建Fiber即可
    }
    // 遍历旧Fiber创建set，键为key值，值为Fiber
    var existingChildren = mapRemainingChildren(returnFiber, oldFiber);
	// 第二次遍历，对于每一个newChild，直接从set里找有没有key值相同可以复用的Fiber
    for (; newIdx < newChildren.length; newIdx++) {
        // updateFromMap 直接去上面生成的map里面去找有没有key值相同的Fiber，如果有直接复用，否则直接新建一个
        var _newFiber2 = updateFromMap(existingChildren, returnFiber, newIdx, newChildren[newIdx], lanes);
	    // 删除map中的oldFiber
        if (_newFiber2 !== null) {
            if (shouldTrackSideEffects) {
                if (_newFiber2.alternate !== null) {
                    existingChildren.delete(_newFiber2.key === null ? newIdx : _newFiber2.key);
                }
            }
		   // 更新lastPlacedIndex
            lastPlacedIndex = placeChild(_newFiber2, lastPlacedIndex, newIdx);
		  // 和第一次遍历时的操作一样
            if (previousNewFiber === null) {
                resultingFirstChild = _newFiber2;
            } else {
                previousNewFiber.sibling = _newFiber2;
            }

            previousNewFiber = _newFiber2;
        }
    }
	// 删除没有用到的旧ChildFiber
    if (shouldTrackSideEffects) {
        existingChildren.forEach(function (child) {
            return deleteChild(returnFiber, child);
        });
    }

    return resultingFirstChild;
}
```

- **两次遍历**
  - **第一次同步遍历找可复用节点**
  - **第二次利用`Set`找可复用节点**

### placeChild

- **用于比较`newFiber`和`oldFiber`的`index`，来确定`newFiber`具体的DOM操作**
- 代码解析如下：

```typescript
function placeChild(newFiber, lastPlacedIndex, newIndex) {
    newFiber.index = newIndex; // 设置newFiber的index
    if (!shouldTrackSideEffects) { // 初始化时不需要标记节点是否需要移动
        return lastPlacedIndex;
    }
    var current = newFiber.alternate; // oldFiber
    if (current !== null) {
        var oldIndex = current.index; // oldFiber下标
        if (oldIndex < lastPlacedIndex) {
            // 旧节点在最新节点下标左边，说明在新DOM中需要右移
            newFiber.flags = Placement; // 标记
            return lastPlacedIndex;
        } else {
            // 返回覆盖lastPlacedIndex
            return oldIndex; // 这个节点不需要移动，在更新后的DOM树中保持原位即可(因为React中同级DOM节点Diff时只右移)，标记下这是新DOM中最新Fiber的index
        }
    } else {
        // 插入操作
        newFiber.flags = Placement;
        return lastPlacedIndex;
    }
}
```

### commitMutationEffects

- **`commitRoot`阶段用于更新真实DOM的函数，看下`Diff`阶段标记Fiber之后，真实DOM怎么操作**
- **`nextEffect`构成一个更新链（所有需要操作的Fiber会形成一个链表，一个接一个处理），还没搞懂这个根据什么顺序连接起来......**
- 已省略部分代码

```typescript
function commitMutationEffects(root, renderPriorityLevel) {
  while (nextEffect !== null) {
    setCurrentFiber(nextEffect); // nextEffect 就是需要更新的Fiber
    var flags = nextEffect.flags; // flags Fiber的标记，比如placeChild设置的移动标记
    var primaryFlags = flags & (Placement | Update | Deletion | Hydrating); // 计算flags
    switch (primaryFlags) {
      case Placement:
        {
          commitPlacement(nextEffect); // 移动节点的函数
          nextEffect.flags &= ~Placement;
          break;
        }
      case PlacementAndUpdate:
        // ...
      case Hydrating:
       	// ...
      case HydratingAndUpdate:
        // ...
      case Update:
        // ...
      case Deletion:
        // ...
    }
    resetCurrentFiber();
    nextEffect = nextEffect.nextEffect; // 获取下一个需要更新的Fiber
  }
}
```

### commitPlacement

- **`commitPlacement`函数内部真正调用DOM操作处理DOM节点**
- 已省略部分代码

```javascript
function commitPlacement(finishedWork) {
  var parentFiber = getHostParentFiber(finishedWork); // 获取父级DOM，本DEMO中就是 ul 节点
  var parent;
  var isContainer;
  var parentStateNode = parentFiber.stateNode; // 获取父节点的真实DOM

  switch (parentFiber.tag) { //根据父节点类型，设置变量
    case HostComponent: // 本DEMO中会走这个
      parent = parentStateNode;
      isContainer = false;
      break;
    case HostRoot:
      //...
    case HostPortal:
      //...
    case FundamentalComponent:
      //...
    default:
      // ... 没找到合适的节点类型而报错
  }

  var before = getHostSibling(finishedWork); // 获取Fiber的sibling链中第一个flags不是Placement的节点(即不需要移动的节点)
  if (isContainer) {
    insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
  } else {
    // 本DEMO调用这个函数
    insertOrAppendPlacementNode(finishedWork, before, parent);
  }
}
```

- **最后的`insertOrAppendPlacementNode`会根据`before`变量是否存在执行以下操作：**
  - **`before`存在，直接调用`parent.insertBefore(current,before)`将`current`放到`before`之前**
  - **`before`不存在，直接调用`parent.appendChild(current)`**



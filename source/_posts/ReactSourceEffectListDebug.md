---
title: ReactSourceEffectListDebug
date: 2022-05-04 14:07:47
tags:
- React
categories:
- React
description: React 源码中关于EffectList的一次学习 
top_img: https://pic2.zhimg.com/v2-8c3223d9228013e069df6a302e81c38f_720w.png?source=d16d100b
cover: /img/cover_9.webp

---
## 前言

### 问题

- React对Fiber树进行遍历标记更新之后，需要在**真实DOM中进行实际操作**，已经知道会在`commit`阶段，那么React是怎么找到需要更新的Fiber的呢？
  - 从Fiber树开始从头遍历会使得性能很低

### 解决方案

- **React在对Fiber进行修改的时候，会同时维护一个特殊的`EffectList`用来保存需要修改的Fiber**
  - `EffectList`就是一个链表，每一个节点的值是一个Fiber
  - 每个Fiber都有自己的`firstEffect`、`lastEffect`，也就是链表的头节点和尾节点

### 学习实例

- DEMO如下（还是Diff源码学习中的案例）

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

## EffectList

### DEOM更新状态

- DEMO运行点击按钮后将要更新的状态，关键更新如下
  - e序列移动到第一个位置
  - 删除c序列

{% asset_img 111.png DEMO更新状态 %}

### EffectList执行时机

- 最后在`commit`阶段，修改真实DOM时有这样的代码（已省略部分代码）
  - **就是去循环获取`EffectList`，根据每一个`Effect`的`flags`去决定修改方式**
  - **会通过FiberRoot的`firstEffect`获取第一个effect，然后进入循环**
  - **每一个Effect都是一个Fiber**
  - **一个典型的空间换取时间的优化机制**

```typescript
while (nextEffect !== null) {
    setCurrentFiber(nextEffect); // 设置Fiber环境
    var flags = nextEffect.flags; // 更新flags
    var primaryFlags = flags & (Placement | Update | Deletion | Hydrating); // 计算flags
    switch (primaryFlags) {
        case Placement: // 移动节点的effect操作
            {
                commitPlacement(nextEffect);ct.flags &= ~Placement;
                break;
            }
    }
    resetCurrentFiber(); // 重置Fiber环境
    nextEffect = nextEffect.nextEffect; // 获取下一个effect，
}
```

### EffectList实例

- 案例更新状态的`EffectList`如下

{% asset_img 222.png EffectList链表 %}

- **即`c->a->b->d->button`这个`effectlist`在这次更新状态中生成，`commit`阶段遍历这个链表，从而一个一个更新真实DOM**

- **可以看到没有`e`节点在`EffectList`中如同之前Diff中说过的，React对节点只进行右移操作，所以`e`节点保持原位即可**

### EffectList生成机制

- **主要继承逻辑就在`completeWork`代码中，即Fiber节点的`complete`阶段**

  ```typescript
  if (returnFiber !== null &&
      (returnFiber.flags & Incomplete) === NoFlags) {
      if (returnFiber.firstEffect === null) { // 父Fiber不存在EffectList时
          returnFiber.firstEffect = completedWork.firstEffect; // 直接继承子Fiber的EffectList
      }
  
      if (completedWork.lastEffect !== null) { // 子Fiber存在EffectList
          if (returnFiber.lastEffect !== null) { // 父Fiber也存在EffectList
              returnFiber.lastEffect.nextEffect = completedWork.firstEffect; // 让父Fiber的EffectList的最后一个Effect连接子Fiber的Effect，实际上就是子链表接到父链表后面
          }
  
          returnFiber.lastEffect = completedWork.lastEffect; // 保证父链表的尾指针指向最后一个Effect
      }
      var flags = completedWork.flags; // 获取当前Fiber的更新标识（flags）
      if (flags > PerformedWork) { // 如果是需要操作DOM的更新
          // 把当前Fiber添加到父Fiber的EffectList中
          if (returnFiber.lastEffect !== null) {
              returnFiber.lastEffect.nextEffect = completedWork;
          } else {
              returnFiber.firstEffect = completedWork;
          }
          // 设置父Fiber的尾指针
          returnFiber.lastEffect = completedWork;
      }
  }
  ```

- **`PerformedWork`的值是1，而不需要变动的Fiber所标识的flags是0（例如本例的节点`e`），其余都会标识都会大于`PerformedWork`这样就把特定的Fiber连接到了`EffectList`中**

- **`a b d`节点的flags早在父Fiber在Diff孩子节点的时候就已经标记了，具体如下**

  ```typescript
  if (oldIndex < lastPlacedIndex) {
      newFiber.flags = Placement; // 标记为移动
      return lastPlacedIndex;
  } else 
      return oldIndex;
  }
  // 没错，就是上个Diff源码学习中的PlaceChild函数(检查节点是否需要移动)中执行的操作
  ```

- **`c`节点也会在Diff时标记，并且直接计入`EffectList`，具体如下**

  ```typescript
  var last = returnFiber.lastEffect;
  last.nextEffect = childToDelete;
  returnFiber.lastEffect = childToDelete;
  // 就在Diff操作的最后阶段执行，用于标记需要删除的childFiber
  ```

- **由于Fiber树的遍历以及完成机制，`completeWork`到FiberRoot时，也就完成了整个`EffectList`，这样在`commit`阶段只需要获取FiberRoot的`firstEffect`，即可开始遍历`EffectList`进行更新操作**

  - 不属于真实DOM的Effect也是同样的机制（例如：`useEffect`注册的函数）

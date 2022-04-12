---
title: vue2SourceDebug
date: 2021-11-15 09:29:39
tags:
- Vue
- source
categories:
- Vue
description: Vue2源码的一次debug
cover: /img/cover_6.webp
top_image: https://pic2.zhimg.com/v2-db7221c0d7ca752b2f88d7ca94939976_1440w.jpg?source=172ae18b

---
## 前言

之前了解Vue源码都是通过网上一些文章获取，从未真正debug过Vue源码，所以就有了这篇文章

**写完之后发现就是源码的堆积，理解还是不够啊**

- Vue版本：2.6.14
- 简单手动配置webpack后进行debug
- 入口文件代码如下

```javascript
/* index.js */
import Vue from "vue";
import Header from "./components/Header/index.js";

debugger; //此处进行debug调试

export default new Vue({
  el: "#app",
  data() { // 测试data
    return {
      text: "123",
      ifFlag: true,
      inputValue: "input something",
    };
  },
  methods: { // 测试methods
    hello() {
      console.log("==app hello");
    },
  },
  computed: { // 测试computed
    reversedText() {
      return this.text.split("").reverse().join("");
    },
  },
  // 测试lifeCycle
  beforeCreate() {
    console.log("==app before create");
  },
  created() {
    console.log("==app created");
  },
  beforeMount() {
    console.log("==app before mount");
  },
  mounted() {
    console.log("==app mounted");
  },
  beforeUpdate() {
    console.log("==app before update");
  },
  updated() {
    console.log("==app updated");
  },
  // 测试component
  components: {
    Header,
  },
  // 测试template
  template: `
    <div id="app" @click="hello">
      <div>{{reversedText}}</div>
      <div v-if="ifFlag">ifFlag is true then I will show</div>
      <input v-model="inputValue" />
      <p>what you input is:{{inputValue}}</p>
      <Header :propValue="'from app'" />
    </div>
  `,
});
```

- Header组件

```javascript
/* Header.js */
import Vue from "vue";

export default Vue.component("Header", {
  template: `
  <div @click="showMsg" class="header">
    text:{{text}}
    propValue:{{propValue}}
  </div>
  `,
  data() {
    return {
      text: "i am header",
    };
  },
  props: ["propValue"], // props测试
  methods: {
    showMsg() {
      console.log("==from header");
    },
  },
  created() {
    console.log("==Header components created");
  },
  mounted() {
    console.log("==Header components mounted");
  },
});
```

## Vue2源码的一次debug

**debug过程中，相关源码会进行简化，省略无关代码（例如所有环境分支均已删除，保留开发环境）**

单步调试`export default new Vue({...})`，就会进入Vue的构造函数内部

```javascript
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword');
  }
  this._init(options);
}
```

- 可以看到Vue的构造函数很简单，接受传入的options，直接调用`_init`方法初始化

### __init

_init方法在源码中已被mixin（混入）Vue构造方法

```javascript
Vue.prototype._init = function (options) {
    var vm = this;

    // a flag to avoid this being observed
    vm._isVue = true;
    // merge options
    if (options && options._isComponent) {
        // optimize internal component instantiation
        // since dynamic options merging is pretty slow, and none of the
        // internal component options needs special treatment.
        initInternalComponent(vm, options);
    } else {
        vm.$options = mergeOptions(
            resolveConstructorOptions(vm.constructor),
            options || {},
            vm
        );
    }
    // expose real self
    vm._self = vm;
    initLifecycle(vm);
    initEvents(vm);
    initRender(vm);
    callHook(vm, 'beforeCreate');
    initInjections(vm); // resolve injections before data/props
    initState(vm);
    initProvide(vm); // resolve provide after data/props
    callHook(vm, 'created');

    if (vm.$options.el) {
        vm.$mount(vm.$options.el);
    }
};
```

- 初始化变量 vm = 当前即将创建的Vue对象
- 标记当前即将创建的对象是Vue`_isVue = true`
- **可以关注下两个声明周期函数的执行时间**
- **最后把Vue挂载到DOM上**
- **下面是每个步骤函数的简单解析**

#### initInternalComponent

- Vnode中存在自定义组件时，在创建真实DOM元素的时候才会调用这个函数

```javascript
function initInternalComponent (vm, options) {
    var opts = vm.$options = Object.create(vm.constructor.options);
    // doing this because it's faster than dynamic enumeration.
    var parentVnode = options._parentVnode;
    opts.parent = options.parent;
    opts._parentVnode = parentVnode;

    var vnodeComponentOptions = parentVnode.componentOptions;
    opts.propsData = vnodeComponentOptions.propsData;
    opts._parentListeners = vnodeComponentOptions.listeners;
    opts._renderChildren = vnodeComponentOptions.children;
    opts._componentTag = vnodeComponentOptions.tag;

    if (options.render) {
        opts.render = options.render;
        opts.staticRenderFns = options.staticRenderFns;
    }
}
```

- 注意：`options.__parentVnode`指的是组件自身的VNode，`options.parentVnode`指的是父组件的Vue实例对象

#### mergeOptoins

- 合并option（选项）

- resolveConstructorOptions函数获取了`Vue.constructor`上的属性

```javascript
  function mergeOptions (
    parent, // Vue.constructor上的属性
    child, // 我们传入的option
    vm // 当前Vue对象
  ) {
    if (process.env.NODE_ENV !== 'production') {
      checkComponents(child);
    }
  
    normalizeProps(child, vm);
    normalizeInject(child, vm);
    normalizeDirectives(child);
  
    // Apply extends and mixins on the child options,
    // but only if it is a raw options object that isn't
    // the result of another mergeOptions call.
    // Only merged options has the _base property.
    if (!child._base) {
      if (child.extends) {
        parent = mergeOptions(parent, child.extends, vm);
      }
      if (child.mixins) {
        for (var i = 0, l = child.mixins.length; i < l; i++) {
          parent = mergeOptions(parent, child.mixins[i], vm);
        }
      }
    }
  
    var options = {};
    var key;
    for (key in parent) {
      mergeField(key);
    }
    for (key in child) {
      if (!hasOwn(parent, key)) {
        mergeField(key);
      }
    }
    function mergeField (key) {
      var strat = strats[key] || defaultStrat;
      options[key] = strat(parent[key], child[key], vm, key);
    }
    return options
  }
  ```

  - `checkComponents`函数检测**自定义组件名是否规范**（例如：不能自定义一个div组件，html已有div标签）
  - `normalizeProps`函数检测**`props`是否规范**（例如：是否是array或者plainObject）
    - 同时也进行了格式化，连字符格式转为驼峰格式
  - normalizeInject、normalizeDirectives函数同上
  - 后续针对extends和mixins方法创建的子构造器合并了选项
  - 最后合并父子的option选项，合并时的优先级如下：（**优先级高的覆盖优先级低的，不会∪在一起**）
    - 自定义option
    - 子option
    - 父option

#### initProxy

- 仅用于开发环境，利用es6的proxy代理Vue属性的访问，当访问不存在的属性时，log提醒

#### initLifecycle

```javascript
function initLifecycle (vm) {
  var options = vm.$options;

  // locate first non-abstract parent
  var parent = options.parent;
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent;
    }
    parent.$children.push(vm);
  }

  vm.$parent = parent;
  vm.$root = parent ? parent.$root : vm; // 当前组件树的根 Vue 实例

  vm.$children = []; // 当前实例的直接子组件
  vm.$refs = {};

  vm._watcher = null;
  // lifeCycle的flag
  vm._inactive = null;
  vm._directInactive = false;
  vm._isMounted = false;
  vm._isDestroyed = false;
  vm._isBeingDestroyed = false;
}
```

- 首先找到第一个非抽象父组件（keep-alive就是抽象组件）
- 初始化vm的一些属性

#### initEvents

```javascript
function initEvents (vm) {
  vm._events = Object.create(null);
  vm._hasHookEvent = false;
  // init parent attached events
  var listeners = vm.$options._parentListeners;
  if (listeners) {
    updateComponentListeners(vm, listeners);
  }
}
```

- 初始化Vue对象的events（事件）功能，包括但不限于：
  - **处理父组件传递的事件**
  - **`this.$on`方法注册事件**
  - **`this.$emit`方法触发事件**

#### initRender

```javascript
function initRender (vm) {
  vm._vnode = null; // the root of the child tree
  vm._staticTrees = null; // v-once cached trees
  var options = vm.$options;
  var parentVnode = vm.$vnode = options._parentVnode; // the placeholder node in parent tree
  var renderContext = parentVnode && parentVnode.context;
  vm.$slots = resolveSlots(options._renderChildren, renderContext);
  vm.$scopedSlots = emptyObject;
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = function (a, b, c, d) { return createElement(vm, a, b, c, d, false); };
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = function (a, b, c, d) { return createElement(vm, a, b, c, d, true); };

  // $attrs & $listeners are exposed for easier HOC creation.
  // they need to be reactive so that HOCs using them are always updated
  var parentData = parentVnode && parentVnode.data;

  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    defineReactive$$1(vm, '$attrs', parentData && parentData.attrs || emptyObject, function () {
      !isUpdatingChildComponent && warn("$attrs is readonly.", vm);
    }, true);
    defineReactive$$1(vm, '$listeners', options._parentListeners || emptyObject, function () {
      !isUpdatingChildComponent && warn("$listeners is readonly.", vm);
    }, true);
  } else {
    defineReactive$$1(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true);
    defineReactive$$1(vm, '$listeners', options._parentListeners || emptyObject, null, true);
  }
}
```

- 初始化vm的一些属性

  - _vnode指的是Vue对象挂载到DOM的VNode
  - $vnode指的是Vue组件在parent中template中的占位node

- resolveSlots函数解析组件传入的slot，并赋值给$slot

  函数实现如下：

  ```javascript
  function resolveSlots (
    children,
    context
  ) {
    if (!children || !children.length) {
      return {}
    }
    var slots = {};
    for (var i = 0, l = children.length; i < l; i++) {
      var child = children[i];
      var data = child.data;
      // remove slot attribute if the node is resolved as a Vue slot node
      if (data && data.attrs && data.attrs.slot) {
        delete data.attrs.slot;
      }
      // named slots should only be respected if the vnode was rendered in the
      // same context.
      if ((child.context === context || child.fnContext === context) &&
        data && data.slot != null
      ) {
        var name = data.slot;
        var slot = (slots[name] || (slots[name] = []));
        if (child.tag === 'template') { // 这里对template进行了特殊处理
          slot.push.apply(slot, child.children || []);
        } else {
          slot.push(child);
        }
      } else {
        (slots.default || (slots.default = [])).push(child);
      }
    }
    // ignore slots that contains only whitespace
    for (var name$1 in slots) {
      if (slots[name$1].every(isWhitespace)) {
        delete slots[name$1];
      }
    }
    return slots
  }
  ```

- 同时创建了$createElement函数（_c函数同理），用于VDOM生成实际DOM

- 最后对`$attrs`和`$listeners`进行了响应式处理

#### callHook('beforeCreated')

顾名思义，此时调用了`beforeCreated`生命周期函数

```javascript
function callHook (vm, hook) {
  // #7573 disable dep collection when invoking lifecycle hooks
  pushTarget();
  var handlers = vm.$options[hook];
  var info = hook + " hook";
  if (handlers) {
    for (var i = 0, j = handlers.length; i < j; i++) {
      invokeWithErrorHandling(handlers[i], vm, null, vm, info);
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook);
  }
  popTarget();
}
```

- 函数头尾禁止了Dep的依赖收集
- `handlers`是一个数组，所以生命周期函数可以是多个
- **invokeWithErrorHandling函数真正执行生命周期函数**

#### initInjections

```javascript
function initInjections (vm) {
  var result = resolveInject(vm.$options.inject, vm);
  if (result) {
    toggleObserving(false);
    Object.keys(result).forEach(function (key) {
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production') {
        defineReactive$$1(vm, key, result[key], function () {
          warn(
            "Avoid mutating an injected value directly since the changes will be " +
            "overwritten whenever the provided component re-renders. " +
            "injection being mutated: \"" + key + "\"",
            vm
          );
        });
      } else {
        defineReactive$$1(vm, key, result[key]);
      }
    });
    toggleObserving(true);
  }
}
```

- 首先找到是谁provide的属性

  ```javascript
  function resolveInject (inject, vm) {
    if (inject) {
      // inject is :any because flow is not smart enough to figure out cached
      var result = Object.create(null);
      var keys = hasSymbol
        ? Reflect.ownKeys(inject)
        : Object.keys(inject);
  
      for (var i = 0; i < keys.length; i++) {
        var key = keys[i];
        // #6574 in case the inject object is observed...
        if (key === '__ob__') { continue }
        var provideKey = inject[key].from;
        var source = vm;
        while (source) {
          if (source._provided && hasOwn(source._provided, provideKey)) {
            result[key] = source._provided[provideKey];
            break
          }
          source = source.$parent;
        }
        if (!source) {
          if ('default' in inject[key]) {
            var provideDefault = inject[key].default;
            result[key] = typeof provideDefault === 'function'
              ? provideDefault.call(vm)
              : provideDefault;
          } else if (process.env.NODE_ENV !== 'production') {
            warn(("Injection \"" + key + "\" not found"), vm);
          }
        }
      }
      return result
    }
  }
  ```

  - 找到所有inject的key值，然后遍历父组件找到对应的provide，获取表达式（值）

- **然后对获取的inject值进行响应式处理（defineReactive$$1）**

#### initState

```javascript
function initState (vm) {
  vm._watchers = [];
  var opts = vm.$options;
  if (opts.props) { initProps(vm, opts.props); }
  if (opts.methods) { initMethods(vm, opts.methods); }
  if (opts.data) {
    initData(vm);
  } else {
    observe(vm._data = {}, true /* asRootData */);
  }
  if (opts.computed) { initComputed(vm, opts.computed); }
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch);
  }
}
```

- initState函数中初始化了我们在option中常用的功能
  - 父组件传值：props
  - 方法：methods
  - 数据：data
  - 计算属性：computed
  - 侦听器：watch

##### initProps

```javascript
function initProps (vm, propsOptions) {
  var propsData = vm.$options.propsData || {};
  var props = vm._props = {};
  // cache prop keys so that future props updates can iterate using Array
  // instead of dynamic object key enumeration.
  var keys = vm.$options._propKeys = [];
  var isRoot = !vm.$parent;
  // root instance props should be converted
  if (!isRoot) {
    toggleObserving(false);
  }
  var loop = function ( key ) {
    keys.push(key);
    var value = validateProp(key, propsOptions, propsData, vm);
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      var hyphenatedKey = hyphenate(key);
      if (isReservedAttribute(hyphenatedKey) ||
          config.isReservedAttr(hyphenatedKey)) {
        warn(
          ("\"" + hyphenatedKey + "\" is a reserved attribute and cannot be used as component prop."),
          vm
        );
      }
      defineReactive$$1(props, key, value, function () {
        if (!isRoot && !isUpdatingChildComponent) {
          warn(
            "Avoid mutating a prop directly since the value will be " +
            "overwritten whenever the parent component re-renders. " +
            "Instead, use a data or computed property based on the prop's " +
            "value. Prop being mutated: \"" + key + "\"",
            vm
          );
        }
      });
    } else {
      defineReactive$$1(props, key, value);
    }
    // static props are already proxied on the component's prototype
    // during Vue.extend(). We only need to proxy props defined at
    // instantiation here.
    if (!(key in vm)) {
      proxy(vm, "_props", key);
    }
  };

  for (var key in propsOptions) loop( key );
  toggleObserving(true);
}
```

- 循环遍历props的值

- validateProp函数检测props属性的值并做处理

  ```javascript
  function validateProp (
    key,
    propOptions,
    propsData,
    vm
  ) {
    var prop = propOptions[key];
    var absent = !hasOwn(propsData, key);
    var value = propsData[key];
    // boolean casting
    var booleanIndex = getTypeIndex(Boolean, prop.type);
    if (booleanIndex > -1) {
      if (absent && !hasOwn(prop, 'default')) {
        value = false;
      } else if (value === '' || value === hyphenate(key)) {
        // only cast empty string / same name to boolean if
        // boolean has higher priority
        var stringIndex = getTypeIndex(String, prop.type);
        if (stringIndex < 0 || booleanIndex < stringIndex) {
          value = true;
        }
      }
    }
    // check default value
    if (value === undefined) {
      value = getPropDefaultValue(vm, prop, key);
      // since the default value is a fresh copy,
      // make sure to observe it.
      var prevShouldObserve = shouldObserve;
      toggleObserving(true);
      observe(value);
      toggleObserving(prevShouldObserve);
    }
    if (
      process.env.NODE_ENV !== 'production' &&
      // skip validation for weex recycle-list child component props
      !(false)
    ) {
      assertProp(prop, key, value, vm, absent);
    }
    return value
  }
  ```

  - **根据props的type处理默认值**
  - **确保props的响应式**
  - **验证传入的props**

- 最后进行一次响应式处理

##### initMethods

```javascript
function initMethods (vm, methods) {
  var props = vm.$options.props;
  for (var key in methods) {
    if (process.env.NODE_ENV !== 'production') {
      if (typeof methods[key] !== 'function') {
        warn(
          "Method \"" + key + "\" has type \"" + (typeof methods[key]) + "\" in the component definition. " +
          "Did you reference the function correctly?",
          vm
        );
      }
      if (props && hasOwn(props, key)) {
        warn(
          ("Method \"" + key + "\" has already been defined as a prop."),
          vm
        );
      }
      if ((key in vm) && isReserved(key)) {
        warn(
          "Method \"" + key + "\" conflicts with an existing Vue instance method. " +
          "Avoid defining component methods that start with _ or $."
        );
      }
    }
    vm[key] = typeof methods[key] !== 'function' ? noop : bind(methods[key], vm);
  }
}
```

- 做了写简单的验证，之后通过bind函数绑定this，然后挂载到vm（Vue实例）上

##### initData

```javascript
function initData (vm) {
  var data = vm.$options.data;
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {};
  if (!isPlainObject(data)) {
    data = {};
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    );
  }
  // proxy data on instance
  var keys = Object.keys(data);
  var props = vm.$options.props;
  var methods = vm.$options.methods;
  var i = keys.length;
  while (i--) {
    var key = keys[i];
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          ("Method \"" + key + "\" has already been defined as a data property."),
          vm
        );
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        "The data property \"" + key + "\" is already declared as a prop. " +
        "Use prop default value instead.",
        vm
      );
    } else if (!isReserved(key)) {
      proxy(vm, "_data", key);
    }
  }
  // observe data
  observe(data, true /* asRootData */);
}
```

- 获取data数据，如果是函数，通过getData函数获取里面的值
- 进行一些简单的验证
  - 方法重名、props重名、data函数没有返回plainObje
- proxy代理`_data`属性
- **响应式处理data**

##### initComputed

```javascript
function initComputed (vm, computed) {
  // $flow-disable-line
  var watchers = vm._computedWatchers = Object.create(null);
  // computed properties are just getters during SSR
  var isSSR = isServerRendering();

  for (var key in computed) {
    var userDef = computed[key];
    var getter = typeof userDef === 'function' ? userDef : userDef.get;
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        ("Getter is missing for computed property \"" + key + "\"."),
        vm
      );
    }

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      );
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef);
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(("The computed property \"" + key + "\" is already defined in data."), vm);
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(("The computed property \"" + key + "\" is already defined as a prop."), vm);
      } else if (vm.$options.methods && key in vm.$options.methods) {
        warn(("The computed property \"" + key + "\" is already defined as a method."), vm);
      }
    }
  }
}
```

- 首先创建了一个computed专用的watcher数组
- 遍历每一个computed，同时为他们每一个创建一个watcher
- 最后defineComputed函数则把computed挂载到vm上，内部实际使用的是watcher的方法

##### initWatch

```javascript
function initWatch (vm, watch) {
  for (var key in watch) {
    var handler = watch[key];
    if (Array.isArray(handler)) {
      for (var i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i]);
      }
    } else {
      createWatcher(vm, key, handler);
    }
  }
}
```

- 遍历用户的watcher创建，内部使用`Vue.$watch`方法
- `Vue.$watch`方法则是使用watcher（观察者实现）

#### initProvide

```javascript
function initProvide (vm) {
  var provide = vm.$options.provide;
  if (provide) {
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide;
  }
}
```

- 存在provide就挂载到vm上

#### callHook('created')

- 和之前的生命周期函数相同，此时调用`created`生命周期函数

### $mount

- `_init`函数的最后一步（如果存在el属性）

```javascript
Vue.prototype.$mount = function (
  el,
  hydrating
) {
  el = el && query(el);

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      "Do not mount Vue to <html> or <body> - mount to normal elements instead."
    );
    return this
  }

  var options = this.$options;
  // resolve template/el and convert to render function
  if (!options.render) {
    var template = options.template;
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template);
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              ("Template element not found or is empty: " + (options.template)),
              this
            );
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML;
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this);
        }
        return this
      }
    } else if (el) {
      template = getOuterHTML(el);
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile');
      }

      var ref = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines: shouldDecodeNewlines,
        shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this);
      var render = ref.render;
      var staticRenderFns = ref.staticRenderFns;
      options.render = render;
      options.staticRenderFns = staticRenderFns;

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end');
        measure(("vue " + (this._name) + " compile"), 'compile', 'compile end');
      }
    }
  }
  return mount.call(this, el, hydrating)
};
```

- 首先使用`query`函数查询`el`指定的DOM节点

- **然后检查`template`并将模板编译为渲染函数（render）**

  - **模板编译函数：compileToFunctions**
    - **内部会将template处理为ast，然后转换为`$createElement`函数可以处理的VDOM，`$createElement`处理之后就会生成VNode**
  - **这也是Vue可以支持JSX写法的原因**

- 最后调用mount函数

  ```javascript
  // public mount method
  Vue.prototype.$mount = function (
    el,
    hydrating
  ) {
    el = el && inBrowser ? query(el) : undefined;
    return mountComponent(this, el, hydrating)
  };
  ```

  - 注意这个mount和之前的$mount函数不同

### mountComponent

```javascript
function mountComponent (
  vm,
  el,
  hydrating
) {
  vm.$el = el;
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode;
    if (process.env.NODE_ENV !== 'production') {
      /* istanbul ignore if */
      if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
        vm.$options.el || el) {
        warn(
          'You are using the runtime-only build of Vue where the template ' +
          'compiler is not available. Either pre-compile the templates into ' +
          'render functions, or use the compiler-included build.',
          vm
        );
      } else {
        warn(
          'Failed to mount component: template or render function not defined.',
          vm
        );
      }
    }
  }
  callHook(vm, 'beforeMount');

  var updateComponent;
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = function () {
      var name = vm._name;
      var id = vm._uid;
      var startTag = "vue-perf-start:" + id;
      var endTag = "vue-perf-end:" + id;

      mark(startTag);
      var vnode = vm._render();
      mark(endTag);
      measure(("vue " + name + " render"), startTag, endTag);

      mark(startTag);
      vm._update(vnode, hydrating);
      mark(endTag);
      measure(("vue " + name + " patch"), startTag, endTag);
    };
  } else {
    updateComponent = function () {
      vm._update(vm._render(), hydrating);
    };
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, {
    before: function before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate');
      }
    }
  }, true /* isRenderWatcher */);
  hydrating = false;

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true;
    callHook(vm, 'mounted');
  }
  return vm
}
```

- 首先会检测option的render函数是否存在（template编译的或者JSX）
- **接着执行`beforeMounted`生命周期函数**
- **创建`updateComponent`函数，内部使用`vm._udpate(vm._render)`**
- 以`updateComponent`函数为`exprOrFn`创建watcher
- 最后执行本组件的`mounted`生命周期函数

#### Watcher

```javascript
var Watcher = function Watcher (
  vm,
  expOrFn,
  cb,
  options,
  isRenderWatcher
) {
  this.vm = vm;
  if (isRenderWatcher) {
    vm._watcher = this;
  }
  vm._watchers.push(this);
  // options
  if (options) {
    this.deep = !!options.deep;
    this.user = !!options.user;
    this.lazy = !!options.lazy;
    this.sync = !!options.sync;
    this.before = options.before;
  } else {
    this.deep = this.user = this.lazy = this.sync = false;
  }
  this.cb = cb;
  this.id = ++uid$2; // uid for batching
  this.active = true;
  this.dirty = this.lazy; // for lazy watchers
  this.deps = [];
  this.newDeps = [];
  this.depIds = new _Set();
  this.newDepIds = new _Set();
  this.expression = process.env.NODE_ENV !== 'production'
    ? expOrFn.toString()
    : '';
  // parse expression for getter
  if (typeof expOrFn === 'function') {
    this.getter = expOrFn;
  } else {
    this.getter = parsePath(expOrFn);
    if (!this.getter) {
      this.getter = noop;
      process.env.NODE_ENV !== 'production' && warn(
        "Failed watching path: \"" + expOrFn + "\" " +
        'Watcher only accepts simple dot-delimited paths. ' +
        'For full control, use a function instead.',
        vm
      );
    }
  }
  this.value = this.lazy
    ? undefined
    : this.get();
};
```

- Watcher初始化最后会通过`get`函数获取初始值

- **`get`函数则是通过调用Watcher初始化的getter获取值**

  - 那么最后就是调用`mountComponent`函数中声明的`updateComponent`函数

    ```javascript
    updateComponent = function () {
        vm._update(vm._render(), hydrating);
    };
    ```

#### _render

```javascript
Vue.prototype._render = function () {
    var vm = this;
    var ref = vm.$options;
    var render = ref.render;
    var _parentVnode = ref._parentVnode;

    if (_parentVnode) {
        vm.$scopedSlots = normalizeScopedSlots(
            _parentVnode.data.scopedSlots,
            vm.$slots,
            vm.$scopedSlots
        );
    }

    // set parent vnode. this allows render functions to have access
    // to the data on the placeholder node.
    vm.$vnode = _parentVnode;
    // render self
    var vnode;
    try {
        // There's no need to maintain a stack because all render fns are called
        // separately from one another. Nested component's render fns are called
        // when parent component is patched.
        currentRenderingInstance = vm;
        vnode = render.call(vm._renderProxy, vm.$createElement);
    } catch (e) {
        handleError(e, vm, "render");
        // return error render result,
        // or previous vnode to prevent render error causing blank component
        /* istanbul ignore else */
        if (process.env.NODE_ENV !== 'production' && vm.$options.renderError) {
            try {
                vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e);
            } catch (e) {
                handleError(e, vm, "renderError");
                vnode = vm._vnode;
            }
        } else {
            vnode = vm._vnode;
        }
    } finally {
        currentRenderingInstance = null;
    }
    // if the returned array contains only a single node, allow it
    if (Array.isArray(vnode) && vnode.length === 1) {
        vnode = vnode[0];
    }
    // return empty vnode in case the render function errored out
    if (!(vnode instanceof VNode)) {
        if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
            warn(
                'Multiple root nodes returned from render function. Render function ' +
                'should return a single root node.',
                vm
            );
        }
        vnode = createEmptyVNode();
    }
    // set parent
    vnode.parent = _parentVnode;
    return vnode
};
```

- **`_render`函数主要通过传入`$createElement`函数生成VNode，并返回**

#### _update

```javascript
  Vue.prototype._update = function (vnode, hydrating) {
    var vm = this;
    var prevEl = vm.$el;
    var prevVnode = vm._vnode;
    var restoreActiveInstance = setActiveInstance(vm);
    vm._vnode = vnode;
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */);
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode);
    }
    restoreActiveInstance();
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null;
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm;
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el;
    }
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  };
```

- **`_update`函数主要通过patch函数将VNode转化为实际DOM**

#### patch

```javascript
function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
        // 无新VNode，存在旧VNode，删除旧VNode即可
        if (isDef(oldVnode)) { invokeDestroyHook(oldVnode); }
        return
    }

    var isInitialPatch = false;
    var insertedVnodeQueue = [];

    if (isUndef(oldVnode)) {
        // 不存在旧节点，直接根据新VNode创建新节点即可
        // empty mount (likely as component), create new root element
        isInitialPatch = true;
        createElm(vnode, insertedVnodeQueue);
    } else {
        var isRealElement = isDef(oldVnode.nodeType);
        if (!isRealElement && sameVnode(oldVnode, vnode)) {
            // patch existing root node
            patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly);
        } else {
            if (isRealElement) {
                // mounting to a real element
                // check if this is server-rendered content and if we can perform
                // a successful hydration.
                if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
                    oldVnode.removeAttribute(SSR_ATTR);
                    hydrating = true;
                }
                if (isTrue(hydrating)) {
                    if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
                        invokeInsertHook(vnode, insertedVnodeQueue, true);
                        return oldVnode
                    } else if (process.env.NODE_ENV !== 'production') {
                        warn(
                            'The client-side rendered virtual DOM tree is not matching ' +
                            'server-rendered content. This is likely caused by incorrect ' +
                            'HTML markup, for example nesting block-level elements inside ' +
                            '<p>, or missing <tbody>. Bailing hydration and performing ' +
                            'full client-side render.'
                        );
                    }
                }
                // either not server-rendered, or hydration failed.
                // create an empty node and replace it
                oldVnode = emptyNodeAt(oldVnode);
            }

            // replacing existing element
            var oldElm = oldVnode.elm;
            var parentElm = nodeOps.parentNode(oldElm);

            // create new node
            createElm(
                vnode,
                insertedVnodeQueue,
                // extremely rare edge case: do not insert if old element is in a
                // leaving transition. Only happens when combining transition +
                // keep-alive + HOCs. (#4590)
                oldElm._leaveCb ? null : parentElm,
                nodeOps.nextSibling(oldElm)
            );

            // update parent placeholder node element, recursively
            if (isDef(vnode.parent)) {
                var ancestor = vnode.parent;
                var patchable = isPatchable(vnode);
                while (ancestor) {
                    for (var i = 0; i < cbs.destroy.length; ++i) {
                        cbs.destroy[i](ancestor);
                    }
                    ancestor.elm = vnode.elm;
                    if (patchable) {
                        for (var i$1 = 0; i$1 < cbs.create.length; ++i$1) {
                            cbs.create[i$1](emptyNode, ancestor);
                        }
                        // #6513
                        // invoke insert hooks that may have been merged by create hooks.
                        // e.g. for directives that uses the "inserted" hook.
                        var insert = ancestor.data.hook.insert;
                        if (insert.merged) {
                            // start at index 1 to avoid re-invoking component mounted hook
                            for (var i$2 = 1; i$2 < insert.fns.length; i$2++) {
                                insert.fns[i$2]();
                            }
                        }
                    } else {
                        registerRef(ancestor);
                    }
                    ancestor = ancestor.parent;
                }
            }

            // destroy old node
            if (isDef(parentElm)) {
                removeVnodes([oldVnode], 0, 0);
            } else if (isDef(oldVnode.tag)) {
                invokeDestroyHook(oldVnode);
            }
        }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch);
    return vnode.elm
}
```

- **`patch`函数是真正更新DOM的重要函数，根据情况不同，更新方式也不同**
  - **`patchVnode`函数用于更新已存在的DOM，著名的DIFF算法就在其中**

- **`createElm`函数根据VNode创建真实DOM节点，随后插入真实DOM树**
  - **前面进行一些判断简化操作，最后去除oldDOM节点**

#### createElm

```javascript
function createElm (
vnode,
 insertedVnodeQueue,
 parentElm,
 refElm,
 nested,
 ownerArray,
 index
) {
    if (isDef(vnode.elm) && isDef(ownerArray)) {
        // This vnode was used in a previous render!
        // now it's used as a new node, overwriting its elm would cause
        // potential patch errors down the road when it's used as an insertion
        // reference node. Instead, we clone the node on-demand before creating
        // associated DOM element for it.
        vnode = ownerArray[index] = cloneVNode(vnode);
    }

    vnode.isRootInsert = !nested; // for transition enter check
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
        return
    }

    var data = vnode.data;
    var children = vnode.children;
    var tag = vnode.tag;
    if (isDef(tag)) {
        if (process.env.NODE_ENV !== 'production') {
            if (data && data.pre) {
                creatingElmInVPre++;
            }
            if (isUnknownElement$$1(vnode, creatingElmInVPre)) {
                warn(
                    'Unknown custom element: <' + tag + '> - did you ' +
                    'register the component correctly? For recursive components, ' +
                    'make sure to provide the "name" option.',
                    vnode.context
                );
            }
        }

        vnode.elm = vnode.ns
            ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode);
        setScope(vnode);

        /* istanbul ignore if */
        {
            createChildren(vnode, children, insertedVnodeQueue);
            if (isDef(data)) {
                invokeCreateHooks(vnode, insertedVnodeQueue);
            }
            insert(parentElm, vnode.elm, refElm);
        }

        if (process.env.NODE_ENV !== 'production' && data && data.pre) {
            creatingElmInVPre--;
        }
    } else if (isTrue(vnode.isComment)) {
        vnode.elm = nodeOps.createComment(vnode.text);
        insert(parentElm, vnode.elm, refElm);
    } else {
        vnode.elm = nodeOps.createTextNode(vnode.text);
        insert(parentElm, vnode.elm, refElm);
    }
}
```

- **如果是组件会去使用`createComponent`函数创建组件**

- **如果是已知的HTML标签，会直接使用`nodeOps.createElement`函数创建**

  - **`nodeOps.createElement`函数内部使用原生DOM操作**

    ```javascript
    function createElement$1 (tagName, vnode) {
      var elm = document.createElement(tagName);
      if (tagName !== 'select') {
        return elm
      }
      // false or null will remove the attribute but undefined will not
      if (vnode.data && vnode.data.attrs && vnode.data.attrs.multiple !== undefined) {
        elm.setAttribute('multiple', 'multiple');
      }
      return elm
    }
    ```

- **如果存在子节点，则会调用`createChildren`函数创建子节点**

  ```javascript
  function createChildren (vnode, children, insertedVnodeQueue) {
      if (Array.isArray(children)) {
          if (process.env.NODE_ENV !== 'production') {
              checkDuplicateKeys(children);
          }
          for (var i = 0; i < children.length; ++i) {
              createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i);
          }
      } else if (isPrimitive(vnode.text)) {
          nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)));
      }
  }
  ```

  - **递归调用`createElm`函数创建子节点的真实DOM**

- **如果是文本节点或注释节点，则分别使用对应的原生DOM操作去创建**

- **创建完成之后调用`insert`函数插入父节点**

  ```javascript
  function insert (parent, elm, ref$$1) {
      if (isDef(parent)) {
          if (isDef(ref$$1)) {
              if (nodeOps.parentNode(ref$$1) === parent) {
                  nodeOps.insertBefore(parent, elm, ref$$1);
              }
          } else {
              nodeOps.appendChild(parent, elm);
          }
      }
  }
  ```

  - **注意此时父节点可能没有挂载到视图上（例如：根Vue），所以没有在视图上展示**

#### createComponent

```javascript
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    var i = vnode.data;
    if (isDef(i)) {
        var isReactivated = isDef(vnode.componentInstance) && i.keepAlive;
        if (isDef(i = i.hook) && isDef(i = i.init)) {
            i(vnode, false /* hydrating */);
        }
        // after calling the init hook, if the vnode is a child component
        // it should've created a child instance and mounted it. the child
        // component also has set the placeholder vnode's elm.
        // in that case we can just return the element and be done.
        if (isDef(vnode.componentInstance)) {
            initComponent(vnode, insertedVnodeQueue);
            insert(parentElm, vnode.elm, refElm);
            if (isTrue(isReactivated)) {
                reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
            }
            return true
        }
    }
}
```

- 组件的VNode的data存有hook函数，是由`render`函数注入的？componentOptions属性存有组件的`props、tag、listeners、Ctor`（Ctor是VueComponent函数，tag是我们定义的组件名）

- `init`hook函数如下：

  ```javascript
  function init (vnode, hydrating) {
      if (
          vnode.componentInstance &&
          !vnode.componentInstance._isDestroyed &&
          vnode.data.keepAlive
      ) {
          // kept-alive components, treat as a patch
          var mountedNode = vnode; // work around flow
          componentVNodeHooks.prepatch(mountedNode, mountedNode);
      } else {
          var child = vnode.componentInstance = createComponentInstanceForVnode(
              vnode,
              activeInstance
          );
          child.$mount(hydrating ? vnode.elm : undefined, hydrating);
      }
  },
  ```

  - **`createComponentInstanceForVnode`函数会调用VueComponent的构造函数，最后调用Vue.__init函数**
  - **由于没有`el`属性，子组件`init`之后就会返回`VueComponent`对象（不会自动调用`init`的$mount函数进行挂载）**
    - 随后`init`函数进行手动挂载

#### invokeInsertHook

```javascript
function invokeInsertHook (vnode, queue, initial) {
    // delay insert hooks for component root nodes, invoke them after the
    // element is really inserted
    if (isTrue(initial) && isDef(vnode.parent)) {
        vnode.parent.data.pendingInsert = queue;
    } else {
        for (var i = 0; i < queue.length; ++i) {
            queue[i].data.hook.insert(queue[i]);
        }
    }
}
```

- **子组件的`insert`hook都会保留到父组件`patch`函数结尾执行**
  - **`mounted`声明周期函数在此处执行**
  - **`v-model`指令在此处会在DOM节点上绑定相关事件，实现双向绑定**



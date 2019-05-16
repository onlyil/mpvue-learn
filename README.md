[TOC]

### 源码

[mpvue github](https://github.com/Meituan-Dianping/mpvue)

### 入口

在 `build/config.js` 中找到 mpvue 打包配置，entry 便是入口文件
```javascript
'mpvue': {
    mp: true,
    entry: resolve('mp/entry-runtime.js'),
    dest: resolve('packages/mpvue/index.js'),
    format: 'umd',
    env: 'production',
    banner: mpBanner
},
```

找到 `src/platforms/mp/entry-runtime.js`，发现是从另外一个文件导入 `Vue` 并导出
```javascript
import Vue from './runtime/index'

export default Vue
```
找到 `mp/runtime/index.js`，即是真正的入口文件

### 平台化包装

主要关注这里，在 Vue 原型上添加了 `_initMP` 属性
```javascript
// for mp
import { initMP } from './lifecycle'
Vue.prototype._initMP = initMP
```

来到 `./lifecycle`，找到 `initMP` 函数，这个函数有点长，我们慢慢看
首先是这段
```javascript
const rootVueVM = this.$root
if (!rootVueVM.$mp) {
  rootVueVM.$mp = {}
}
```
这里在 `$root` 也就是 `Vue` 根实例上添加 `$mp` 属性，后面会发现它维护了该实例对应的小程序 `App` `Page` 对象的相关信息

接下来是
```javascript
const mp = rootVueVM.$mp
if (mp.status) {
  // 处理子组件的小程序生命周期
  if (mpType === 'app') {
    callHook(this, 'onLaunch', mp.appOptions)
  } else {
    callHook(this, 'onLoad', mp.query)
    callHook(this, 'onReady')
  }
  return next()
}
```
首先判断 `mp.status` 如果存在，说明该实例已经初始化过，继续判断 `mpType` ，根据类型调用不同的生命周期钩子，然后执行转入的 `next`

### 初始化生命周期

接下来是
```javascript
  mp.mpType = mpType
  mp.status = 'register'

  if (mpType === 'app') {
    // ...
  } else if (mpType === 'component') {
    // ...
  } else {
    // ...
  }
```
如果 `mp.status` 不存在，则初始化 `mp.status = 'register'`，然后判断 `mpType` ，根据类型进行不同的初始化操作
首先是 `mpType === 'app'`
```javascript
global.App({
  // 页面的初始数据
  globalData: {
    appOptions: {}
  },

  handleProxy (e) {
    return rootVueVM.$handleProxyWithVue(e)
  },

  // Do something initial when launch.
  onLaunch (options = {}) {
    mp.app = this
    mp.status = 'launch'
    this.globalData.appOptions = mp.appOptions = options
    callHook(rootVueVM, 'onLaunch', options)
    next()
  },
  // ...
})
```
执行了 `global.App()`，这里的 `global` 是哪里定义的呢？
在 `mp/join-code-in-build.js` 中
```javascript
exports.mpBanner = `// fix env
try {
  if (!global) global = {};
  global.process = global.process || {};
  global.process.env = global.process.env || {};
  global.App = global.App || App;
  global.Page = global.Page || Page;
  global.Component = global.Component || Component;
  global.getApp = global.getApp || getApp;

  if (typeof wx !== 'undefined') {
    global.mpvue = wx;
    global.mpvuePlatform = 'wx';
  } else if (typeof swan !== 'undefined') {
    global.mpvue = swan;
    global.mpvuePlatform = 'swan';
  }else if (typeof tt !== 'undefined') {
    global.mpvue = tt;
    global.mpvuePlatform = 'tt';
  }else if (typeof my !== 'undefined') {
    global.mpvue = my;
    global.mpvuePlatform = 'my';
  }
} catch (e) {}
`
```
来到之前 `build/config.js`
```javascript
'mpvue': {
  mp: true,
  entry: resolve('mp/entry-runtime.js'),
  dest: resolve('packages/mpvue/index.js'),
  format: 'umd',
  env: 'production',
  banner: mpBanner
},
```
其中 `banner: mpBanner` 会在打包时注入，从而挂在全局 `global`

回到之前的代码，主要是在各个生命周期函数中修改实例 `mp.status`，小程序 `option` 参数代理访问和调用钩子。以 `onLaunch` 为例，
```javascript
// Do something initial when launch.
onLaunch (options = {}) {
  mp.app = this
  mp.status = 'launch'
  this.globalData.appOptions = mp.appOptions = options
  callHook(rootVueVM, 'onLaunch', options)
  next()
},
```
首先 `mp.app = this`，所以我们在 `App.vue` 中可以通过 `this.$mp.app` 访问小程序 App 对象（**注意 `created` 钩子中不可以，因为在执行 `Vue.prototype.$mount` 中才初始化 `_initMP`，然后才有 `this.$mp`**），**以此可以在小程序 `App` 上挂载自定义数据，如  `skynet、bus` 等**
然后修改 `mp.status` 状态，将参数 options 维护在 `globalData.appOptions`
最后调用钩子 `callHook(rootVueVM, 'onLaunch', options)`

看一下 `callHook` 的代码
```javascript
export function callHook (vm, hook, params) {
  let handlers = vm.$options[hook]
  // ...
  let ret
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      try {
        ret = handlers[i].call(vm, params)
      } catch (e) {
        handleError(e, vm, `${hook} hook`)
      }
    }
  }
  // ...
  // for child
  if (vm.$children.length) {
    vm.$children.forEach(v => callHook(v, hook, params))
  }

  return ret
}
```

首先是这样一行：
```javascript
let handlers = vm.$options[hook]
```
**`vm.$options` 是 `vue` 选项合并的最终结果，从中取到对应的钩子函数数组 `handlers`，然后遍历依次执行，这就解释了为什么 `mpvue` 兼容小程序生命周期**

`callHook` 之后是：
```javascript
next()
```
这个 `next` 是 `initMP` 的一个入参，那么它是什么呢？

回到入口文件 `mp/runtime/index.js`，有段代码定义了 `Vue.prototype.$mount`：
```javascript
Vue.prototype.$mount = function (el, hydrating) {
  // 初始化小程序生命周期相关
  const options = this.$options

  if (options && (options.render || options.mpType)) {
    const { mpType = 'page' } = options
    return this._initMP(mpType, () => {
      return mountComponent(this, undefined, undefined)
    })
  } else {
    return mountComponent(this, undefined, undefined)
  }
}
```
重点看这一段：
```javascript
return this._initMP(mpType, () => {
  return mountComponent(this, undefined, undefined)
})
```
可以得知传入的是一个函数，它执行了 `mountComponent`，其核心是调用 `beforeMounted` 钩子，同时实例化一个渲染函数观察者，进行依赖收集和数据更新。**由此可知，生命周期钩子的调用顺序：`created` -> `onLaunch` -> `beforeMounted`**

至此，对小程序的初始化就结束了，总结来说就是：
1. 在实例上代理访问 `$mp`，维护小程序 app 或 page 或 component 信息
2. 很据 `mpType` 初始化小程序 App 或 Page 生命周期
3. 执行 `mountComponent`，挂载根实例

### 同步数据

#### 初始化

生命周期已经初始化完成，那么下一步就是将 vue 实例上的数据同步到小程序了
这一步是在什么时候做的呢？
在之前的初始化生命周期的代码中可以看到，在页面 `onShow` 和组件 `ready` 中会执行 `initDataToMP`
回到入口文件 `mp/runtime/index.js` 下方有这样一段代码：
```javascript
import { updateDataToMP, initDataToMP } from './render'
Vue.prototype.$updateDataToMP = updateDataToMP
Vue.prototype._initDataToMP = initDataToMP
```
这里定义了两个方法，字面意思好像是更新数据并映射到小程序，初始化数据到小程序，实例上确实是这样的
我们先来看 `initDataToMP`
来到 `mp/runtime/render.js` 找到该函数：
```javascript
export function initDataToMP () {
  const page = getPage(this)
  if (!page) {
    return
  }

  const data = collectVmData(this.$root)
  page.setData(data)
}
```
可以看到主要是调用了 `collectVmData` 得到了初始数据
```javascript
function collectVmData (vm, res = {}) {
  const { $children: vms } = vm
  if (vms && vms.length) {
    vms.forEach(v => collectVmData(v, res))
  }
  return Object.assign(res, formatVmData(vm))
}
```
从页面根实例开始递归处理子组件，这里调用了另外一个方法 `formatVmData`

`formatVmData` 格式化之后的 `data` 是这个样子：
```javascript
{
  $root.0_1: {
    $p: '0',
    $k: '0_1',
    $kk: '0_1_',
    text: 'hello'
  }
}
```
- `$p` 指的是该组件的父组件的标识符
- `$k` 指的是该组件自身的标识符
- `$kk` 指的是该组件自身的标识符前缀

回到 `initDataToMP`，执行 `page.setData(data)` 为笑程序页面初始化数据

#### 更新

接下来看如何更新数据
```javascript
Vue.prototype.$updateDataToMP = updateDataToMP
```

```javascript
export function updateDataToMP () {
  const page = getPage(this)
  if (!page) {
    return
  }

  const data = formatVmData(this)
  diffData(this, data)
  throttleSetData(page.setData.bind(page), data)
}
```
其大体结构同上，只是在 `setData` 之前做了 `diff` 和 节流
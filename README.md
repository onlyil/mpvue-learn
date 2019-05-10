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
找到 `mp/runtime/index.js`，即使真正的入口文件

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
这里在 `$root` 也就是 Vue 根实例上添加 `$mp` 属性，后面会发现它维护了该实例对应的小程序 `App` `Page` 对象的相关信息

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

  // Do something when app show.
  onShow (options = {}) {
    mp.status = 'show'
    this.globalData.appOptions = mp.appOptions = options
    callHook(rootVueVM, 'onShow', options)
  },

  // Do something when app hide.
  onHide () {
    mp.status = 'hide'
    callHook(rootVueVM, 'onHide')
  },

  onError (err) {
    callHook(rootVueVM, 'onError', err)
  },

  onPageNotFound (err) {
    callHook(rootVueVM, 'onPageNotFound', err)
  }
})
```
执行了 `global.App()`，这里的 global 是哪里定义的呢？
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

回到之前的代码，主要是在各个生命周期函数中修改实例 `mp.status`，小程序 option 参数代理访问和调用钩子。以 onLaunch 为例，
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
首先 `mp.app = this`，所以我们在 `App.vue` 中（注意 created 钩子中不可以 TODO: 原因）可以通过 `this.$mp.app` 访问小程序 App 对象，在此可以在小程序 App 上挂载自定义数据，如  skynet、bus 等
然后修改 `mp.status` 状态，将参数 options 维护在 `globalData.appOptions`
最后调用钩子 `callHook(rootVueVM, 'onLaunch', options)`

看一下 callHook 的代码
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
`vm.$options` 是 vue 选项合并的最终结果，从中取到对应的钩子函数数组 handlers，然后遍历依次执行，这就解释了为什么 mpvue 兼容小程序生命周期





### 1. 组件引用时不支持 v-show

#### 例
```html
<!-- 编译前 vue template -->
<card v-show="showCard"></card>

<!-- 编译后 wxml -->
<template hidden="{{!(showCard)}}" data="{{...$root[$kk+'3'], $root}}" is="3b910776"></template>
```

#### 原因
v-show 会被编译为 hidden，但是小程序 template/block 均不支持 hidden，只支持 wx:if

#### 拓展
hidden 的实现原理其实是 css：
```css
view[hidden] {
    display: none;
}
```
- 所以如果一个 view 样式存在 display 时，它的优先级可能会高于 `view[hidden]`，这样 hidden 就失效了
- 这同时也解释了为什么 template/block 均不支持 hidden，template/block 在文档流中均没有节点，样式自然也就不生效

### 2. 组件引用时不支持绑定事件

#### 例
```html
<!-- 编译前 vue template -->
<card @click="clickHanler"></card>

<!-- 编译后 wxml -->
<template bindtap="handleProxy" data-eventid="{{'4'}}" data-comkey="{{$k}}" data="{{...$root[$kk+'3'], $root}}" is="3b910776"></template>
```

#### 原因
小程序 template/block 均不支持事件绑定
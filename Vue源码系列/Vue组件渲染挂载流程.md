## Vue.component Vue.extend 初始化组件 与 组件创建挂载流程

### 组件初始化

`Vue.component(组件名,options:{})` 
位于src/global-api/assets,
内部调用了`Vue.extend`传入`options`获得返回值(组件的构造函数) 并赋值给 Vue.options.components.组件名 

`Vue.extend(options)`
位于src/global-api/extend,

内部创建了组件的构造函数 Sub ,并使 Sub 继承 Vue, **以获得Vue的各种方法**。
原语句`Sub.prototype = Object.create(Super.prototype)`

随后给 将传入的 options 与 Vue实例的 options 合并为 sub.options, 把父级的定义放自己身上
原语句`Sub.options = mergeOptions( Super.options, extendOptions )`
这样做是为了获得 全局的组件 指令和钩子，让其在子组件里也能调用。

再对 sub 进行一些处理，挂载函数属性啥的，总之目的都是让组件也能和Vue一样用
最后返回构造函数

### 组件创建挂载流程

组件初始化在声明的时候就挂载好了，真正创建在渲染的时候。

Vue渲染过程就是将 模板 通过一大堆 正则 解析为AST语法树, 再将AST语法树转换为render函数

再调用render函数转化为虚拟DOM，再调用_update，内部调用patch，将虚拟DOM转化为真实DOM


#### 渲染为虚拟DOM
调用render函数转化为虚拟DOM 过程中, 遇到子组件 会调用 src/core/instance/render 里的 createElement 并传入当前实例 及 子组件tag名 等

实际上用的是src/core/vdom/create-element 下的 `createElement` ,该方法判断不同的元素类型创相应vnode

createElement 发现组件时，首先获取 传入的当前实例的options，通过 options.components.tag 找到前初始化好的构造函数

传给 src/core/vdom/create-component 的 `createComponent` ,

`createComponent`将构造函数作为参数之一  创建Vnode (虚拟DOM) ,并return 。

```js
const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context, // data.hook
    { Ctor, propsData, listeners, tag, children }, // Ctor 构造函数
    asyncFactory
  )
return vnode
```
`createComponent`  的过程中 还添加了 4个 组件生命周期 函数 componentVNodeHooks:{init,prepatch,insert,destroy} 在data.hook上


#### 创建并挂载

_init 初始化完数据后内部调用$mount，$mount 内部调用 的是lifecycle文件里的 mountComponent

mountComponent 内部是先调用 _render 获得 vnode 再 _update 

_update 内部是 调用 `__patch__` , 这个patch 实际上就是 core/vdom/patch 文件里的 createPatchFunction return 的patch



找到patch后就知道，patch 根据不同的 vnode 创建 真实DOM，当找到Vnode 为组件时 会执行该子组件hook上的init方法

init方法会创建子组件实例  `return new vnode.componentOptions.Ctor(options)`

并调用$mount， $mount又会走到 patch 方法，并最终执行 createElm 挂载组件。



**渲染为虚拟DOM时，注意一个问题**
web端直接使用vue时， 直接调用Vue.component()  挂载全局组件 与 Vue实例中创建组件结果相同过程不同。
```js
Vue.component('my-button',{
    template:'<button>我的按钮</button>'
})
new Vue({
    el:'#app',
    components:{
        'my-button':{
            template:'<button>我的按钮</button>'
        }
    }
});
```
前者在初始化时就会被解析为构造函数sub挂载到Vue.options.components.my-button上。(函数)
后者则是直接作为Object没有处理并且同样挂载在Vue.options.components.my-button。(对象)

createComponent对传入参数进行检查，假如是对象，此时同样会调用Vue.extend() 将其变为sub函数，假如本来就是函数，则不变。
所以二者本质都得被调用一遍extend()，转化为sub构造函数




 
## 虚拟dom转化为实际dom的过程

## diff算法，patch函数


## 考虑一下key的作用，不设置key或者key设置为index的问题

## 组件的data为什么要是一个函数

## 父子组件的生命周期调用


## 总结一下父子组件的传参数方式（非源码）


## initEvents 这个函数要看一下 最好手写一个off emit once on方法

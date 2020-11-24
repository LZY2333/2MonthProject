# Vue初次渲染流程

new Vue(options:{}) 本质上是做了两件事:

1.给Vue原型挂载各种属性和方法,初始化Vue构造函数.

2.处理new Vue()时传入的options,初始化Vue实例,创建视图.



## 初始化Vue构造函数

从Vue引用顺序角度讲解,最外层的文件开始解析.



### src/plateforms/web/entry-runtime-with-compiler

重写了`$mount()`方法,在原本的`$mount()`之前,加了一层模板编译,能将模板编译成`render()`函数.

---
采用了 切片编程AOP 或者说 函数劫持 的方式.

此文件拥有运行时编译器，能将模板编译为render函数，相比`entry-runtime`文件，只重写了$mount方法。

在原版的`$mount`之前，依次判断了是否有render，否则判断是否有template，否则创建新的空div

之后调用compileToFunctions，返回render函数，给原版的`$mount`使用.

这里以及下面, 均挂载 在 Vue.prototype, 也就是Vue原型 上.

---



### src/plateforms/web/entry-runtime

1. 挂载了`Vue.prototype.$mount()`方法,内部实际上调用的是`mountComponent()`
2. 扩展了平台(Web)相关的全局 指令 和 组件
3. 挂载了`Vue.prototype.__patch__`方法,实际上就是`patch()`
4. 检测浏览器装没装Vue调试插件,没装的话,就宣传一波..... 鱿鱼须:XP

---

`$mount()`方法 实际就是, `mountComponent()`方法, 内部执行了 `vm._update(vm._render())`,并挂载了组件`Watcher`对象.

`vm._render()`: 将 模板编译成的render函数 传入实时data数据 转化 虚拟DOM
`vm._update()`: 内部调用`patch()`方法,将 虚拟DOM 转化为 真实DOM 并挂载
`patch()`: 虚拟节点对比,diff算法,真实节点挂载,都在里面.
Watcher对象: 响应式核心,检测数据变化,变化时,会再次执行`vm._update(vm._render())`.

这些函数,以后有机会我也会总结一下,这里先介绍个概念,也可以搜掘金里其他文章,讲的都很好.

---



### src/core/index

1. initGlobalAPI(Vue)
2. 设置了3个 服务端渲染相关的属性 和 Vue.version

**`initGlobalAPI()`**

1. 劫持了`Vue` 的 `config` 属性 ,使其无法被修改.
2. 挂载 Vue.util = {warn,extend,mergeOptions,defineReactive},并伴有注释:不懂别瞎用..
3. 挂载 Vue.set 和 Vue.delete 作用是响应式地添加和删除数据
4. 挂载 Vue.nextTick 传入的方法会再下一个事件环再执行,操作DOM会用到.(为什么DOM要下一个时间环才能拿到)
6. 挂载 Vue.use方法, initUse(Vue)
7. 挂载 Vue.mixin方法, initMixin(Vue)
8. 挂载 Vue.extend方法, initExtend(Vue)
9. 挂载 Vue.component/directive/filter注册器方法, initAssetRegisters(Vue)

---
**warn**: 警告处理函数.没详细了解过...
**extend**: 接受两个对象类型的参数`extend(to,_from)`,里面就是一个for循环,将from里的属性遍历全部赋给to.
**mergeOptions**: 也是对象合并,不过这里复杂很多,针对Vue组件各类属性都进行了特殊处理.有时间的话详细讲一下.
**defineReactive**: 将普通数据处理成Vue的响应式数据,里面用到了`Object.defineProperty()`重新属性get set和`Dep`对象收集依赖.

initUse的时候用了函数柯里化的方式,提前把Vue传进去了,这样调用use的时候就不用再传Vue,直接Vue.use(Vuex)

**Vue.use**: 安装插件,内部调用了插件的install方法,如Vuex,Vue-router.
**Vue.mixin**: mixin就是全局注册一个混入，影响注册之后所有创建的每个 Vue 实例,内部调用了mergeOptions.一般制作插件的时候会用到这方法.
**Vue.extend**:


顺便讲一下`New Vue({data:{},mixins:[myMixin],extends:{}})` 不要与 `Vue.mixin`,`Vue.extend`搞混,本质就不是一个东西
**mixins属性**,是将组件实例对象与myMixin合并,用于抽离公共逻辑后 与组件合并,本质上就是一个组件.
**extends属性**,和mixins类似,不同在于执行顺序：extends > mixins > 组件,同时mixins接收数组,extends只接收一个对象.

**Vue.mixin**,是将传入参数与Vue的optoins合并,改变的是Vue构造函数的属性,因而也改变了所有Vue实例.
组件,在父组件中引入组件，相当于在父组件中给出一片独立的空间供子组件使用，然后根据props来传值，但本质上两者是相对独立的。
**Vue.extend**,创建组件.创建了一个继承Vue的所有属性方法的构造函数.用来创建可复用的组件.
**Vue.component**,内部就是extend,只不过这里extend以后将返回的组件注册器放到了Vue属性上
**Vue.directive/filter注册器**: 就是把里面传入的东西直接放到了Vue特定属性上

注意: 这一小节全是挂载再Vue构造函数上,属于静态方法,调用方式是Vue.XXX.而Vue实例不能直接调用.
上一小节全是Vue.prototype.XXX,在Vue实例上可调用,如 new Vue().$mount().

---

## src/core/instance/index

Vue真正核心,Vue本体 Vue构造函数就在这个文件,并调用了五个方法挂载各种函数及属性

### initMixin(Vue)

### stateMixin(Vue)

### eventsMixin(Vue)

### lifecycleMixin(Vue)

### renderMixin(Vue)

### function Vue (options) {}

Vue本体,简单易懂,里面就是一个`this._init`,随后开启了处理options的流程.

大概就是 构建响应式数据,调用生命周期函数,模板解析,初始化渲染,依赖渲染 等等 穿插进行

由下一大节讲解

```js
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```




## `this._init(options)`

(src/core/instance/index.js)

`Vue.prototype._init` 在 initMixin(Vue) 中创建

(src/core/instance/init.js)

1. `vm._uid` 给vue实例编号
2. `mergeOptions()` 合并父子组件参数(没细看，看上去很麻烦)、
3. `initLifecycle(vm)` 初始化生命周期相关属性,父子组件链
4. `initEvents(vm)` 初始化父级组件的事件监听
5. `initRender(vm)` 获取父vnode及执行上下文，初始化插槽，创建组件$createElement方法
6. `callHook(vm, 'beforeCreate')` 调用beforeCreate
7. `initInjections(vm)` 获取上级注入的依赖(没细看)
8. `initState(vm)` 处理 props,methods, data, computed, watch。
9. `initInjections` 初始化依赖注入(没看)
10. `callHook(vm, 'created')` 调用created

## initState(vm)(待细看)

(src/core/instance/state.js)

1.initProps
2.initMethods
3.initData
4.initComputed
5.initWatch


proxy 给target重写get set,并预设好sourceKey,之后向target取值赋值时,自动去下级sourceKey下找。

```js
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```
## initData (待细看)

`initData`结束返回`initState`,`initState`结束返回`this._init(options)`,`this._init(options)`执行到最后一行

此时数据准备完成调用vm.$mount(vm.$options.el)

## vm.$mount(vm.$options.el)

(src/plateforms/web/entry-runtime-with-compiler.js)

切片编程AOP,函数劫持.

此文件拥有运行时编译器，能将模板编译为render函数，相比without-compiler，重写了$mount方法。

在原版的$mount之前，依次判断了是否有render，否则判断是否有template，否则创建新的空div

之后调用compileToFunctions，返回render函数，给原版的$mount使用。

## 原版vm.$mount

(src/plateforms/web/runtime/index.js)

实际上就是调用了mountComponent

(src/core/instance/lifecycle.js)

1.'callHook(vm, 'beforeMount')' 调用了beforeMount
2.'vm._update(vm._render())'    核心语句，调用render函数，渲染页面(待细看)
3.'callHook(vm, 'mounted')'     调用了mounted

## 初始化完成
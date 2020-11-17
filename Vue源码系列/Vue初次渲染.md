# Vue初次渲染流程

new Vue() 本质上是做了两件事，给Vue原型挂载各种函数，和处理传入的options。

## this._init(options)

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
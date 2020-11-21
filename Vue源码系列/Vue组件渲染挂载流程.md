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


 
## 虚拟dom转化为实际dom的过程 与 patch函数 与 Diff算法

_init 里调用 `vm.$mount(vm.$options.el)`

`vm.$mount(vm.$options.el)` 里调用 `mountComponent`

`mountComponent` 里调用 `vm._render()` 获得 vnode 再调用 `vm._update(vnode, hydrating)`

`vm._update()` 中 主要是调用`__patch__` (位于 src/platforms/web/runtime/index ) 

实际上调用的是 `createPatchFunction` (位于 src/core/vdom/patch ) 中return 的 `patch`

`patch`判断当前进来的是组件,还是初次渲染,还是节点更新.节点更新时调用`patchVnode`.

`patchVnode`根据新旧vnode对比决定是否复用原有DOM,修改哪些属性,插入到哪个位置,儿子节点递归创建等等.

里面再调用`createElm()` 创建真实DOM并插入指定位置.


```js
return function patch (oldVnode, vnode, hydrating, removeOnly) {

    // 如果vnode 新节点不存在 但是 oldVnode存在 说明是要销毁 oldVnode
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    // 如果vnode 存在 但是 oldVnode不存在, 说明是需要创建新节点,则调用 createElm创建新节点
    if (isUndef(oldVnode)) { //进行这一步一般是组件.
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)

    // 当vnode和oldVnode都存在时  
    } else {
      const isRealElement = isDef(oldVnode.nodeType)

    // 如果oldVnode不是真实节点，并且vnode和oldVnode是同一节点时，说明是需要比较新旧节点，则调用patchVnode进行patch。
      if (!isRealElement && sameVnode(oldVnode, vnode)) { // 真正patch, diff算法在里面
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {

    //如果oldVnode是真实节点
        if (isRealElement) {  // 进了这里说明是初次渲染
    // 如果oldVnode是元素节点，且含有data-server-rendered属性，移除该属性，设置hydrating为true。
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }

    // 如果hydrating为true，调用hydrate方法，将Virtural DOM与真实DOM进行映射，然后将oldVnode设置为对应的Virtual DOM。
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            } else if (process.env.NODE_ENV !== 'production') {
              warn(
                'The client-side rendered virtual DOM tree is not matching ' +
                'server-rendered content. This is likely caused by incorrect ' +
                'HTML markup, for example nesting block-level elements inside ' +
                '<p>, or missing <tbody>. Bailing hydration and performing ' +
                'full client-side render.'
              )
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode)
        }

    //走到这里有两种情况
    //oldVnode是真实节点时,说明是初次渲染,直接用新节点渲染代替,
    //oldVnode和vnode是完全不同的节点,如div变成span,也是直接用新节点代替,和初次渲染一样.
    //找到oldVnode.elm的父节点，根据vnode创建一个真实的DOM节点，并插入到该父节点中的oldVnode.elm位置。如果组件根节点被替换，遍历更新父节点element。然后移除旧节点。
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // 递归更新该节点的父节点的啥啥属性.
        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }
```



虚拟DOM将转化为真实DOM并挂载.




## diff算法，


## 考虑一下key的作用，不设置key或者key设置为index的问题

## 组件的data为什么要是一个函数

## 父子组件的生命周期调用


## 总结一下父子组件的传参数方式（非源码）


## initEvents 这个函数要看一下 最好手写一个off emit once on方法

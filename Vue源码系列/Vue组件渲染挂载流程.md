## Vue.component Vue.extend 初始化组件 与 组件创建挂载流程

### 组件初始化 (应该改为注册)

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

### 组件创建挂载流程(应该改为初始化为虚拟节点)

组件初始化在声明的时候就挂载好了，真正创建在渲染的时候。

Vue渲染过程就是将 模板 通过一大堆 正则 解析为AST语法树, 再将AST语法树转换为render函数

再调用render函数转化为虚拟DOM，再调用_update，内部调用patch，将虚拟DOM转化为真实DOM


#### 渲染为虚拟DOM()
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


 



## 虚拟dom转化为真实dom的过程 与 patch函数 与 Diff算法

_init 里调用 `vm.$mount(vm.$options.el)`

`vm.$mount(vm.$options.el)` 里调用 `mountComponent`

`mountComponent` 里调用 `vm._render()` 获得 vnode 再调用 `vm._update(vnode, hydrating)`

`vm._update()` 中 主要是调用`__patch__` (位于 src/platforms/web/runtime/index ) 

实际上调用的是 `createPatchFunction` (位于 src/core/vdom/patch ) 中return 的 `patch`

`patch`判断当前进来的是组件,还是初次渲染,还是相同节点更新.相同节点更新时调用`patchVnode`.

`patchVnode`根据新旧vnode对比决定是否复用原有DOM,修改哪些属性,插入到哪个位置,儿子节点递归创建等等.

里面再调用`createElm()` 创建真实DOM并插入指定位置.


组件初始化的时机是patch 初次渲染 中挂载父节点时,会调用`createElm()`,里面判断不同vnode节点类型,创建对应真实节点,
`createElm()`中,`nodeOps.createElement(tag, vnode)`创建父真实节点后,
调用`createChildren()`,对父节点的所有子节点(虚拟节点)循环回调调用`createElm()`,来创建子节点(真实节点).
`createElm()`中调用`createComponent()`检查每个子节点是否为组件类型,如果是,
则调用子组件的初始化方法`isDef(i = i.hook) && isDef(i = i.init)` ,
而组件初始化方法`i.init`中又调用了子组件的构造函数`return new vnode.componentOptions.Ctor(options)`,
new了一个组件实例,并调用了$mount()方法,将其转化为了真实DOM
挂载到了组件节点vnode的componentInstance属性上,然后回到子节点的`createElm()`继续,走到调用其中的`insert()`,
`insert()`,该方法将真实节点插入指定位置,将其添加到父节点下,子节点处理结束,
之后再回到父节点的`createElm()`,走到调用其中的`insert()`,将其添加到根节点,父节点处理结束.

### patch

patch 主要接受两个参数,一个是oldVnode旧虚拟节点,一个是vnode新虚拟节点,根据两个参数不同进行不同操作.

首先过滤**节点销毁**,只传了oldVnode,代表想销毁该节点
其次过滤**组件渲染**,只传了vnode,代表想直接创建新节点,会走这一分支一般是在组件初始化的过程中
组件初始化会调用patch,给组件的实例挂载一个真实DOM,且此时没有挂载到页面上,只在内存中,供后续调用

对组件类型调用同函数上下文下的`createComponent()`
`createComponent()`,调用该组件类型vnode下的`init()`生命周期函数

进入主题,**数据更新**, oldVnode 和 vnode 都传了, 
且二者满足 均为虚拟节点, key相同, tag相同, 均有data属性,代表可以复用旧DOM
则获得资格进行子节点比较,进入真正的diff算法,调用`patchVnode()`

否则,oldVnode 是真实节点(代表**初次渲染**),初次渲染要用创建的新节点取代`<div id="app"></div>`.
或 oldVnode 和 vnode 都传了,但 key, tag, data 不同(代表**节点替换**),说明两个节点有较大不同,
所以这两种情况都是直接销毁旧节点,用新节点替代

patch ,意思是对比两个节点并更新.
要理解diff 最重要的是理解,我们的目标是两个,将旧真实DOM节点变成新真实DOM节点的样子,以及尽可能复用旧节点
```js
return function patch (oldVnode, vnode, hydrating, removeOnly) {

    // 情况1(节点销毁). 如果vnode 新节点没传 只传了 oldVnode ,就 销毁该节点
    if (isUndef(vnode)) {// 
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    // 情况2(组件渲染).如果vnode 新节点传了 但是 oldVnode 没传, 说明是需要创建新节点,则调用 createElm创建新节点
    if (isUndef(oldVnode)) { //进行这一步一般是组件.
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)

    // 当vnode和oldVnode都传了
    } else {
      const isRealElement = isDef(oldVnode.nodeType)

    // 情况3(数据更新). 如果传入的oldVnode是虚拟节点，并且vnode和oldVnode是同一节点时，说明是数据更新
    // 则调用patchVnode进行patch,这里是diff算法存在的地方.
      if (!isRealElement && sameVnode(oldVnode, vnode)) { // 真正patch, diff算法在里面
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {

    //如果oldVnode是真实节点
        if (isRealElement) { // 这个if块里全是服务端渲染的东西,可以忽略.
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }

    
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
    //情况4(初次渲染). oldVnode是真实节点时,说明是初次渲染,直接用新节点渲染代替,
    //情况5(节点替换). oldVnode和vnode是完全不同的节点,如div变成span,也是直接用新节点代替,和初次渲染一样.
    //找到oldVnode.elm的父节点，根据vnode创建一个真实的DOM节点，并插入到该父节点中的oldVnode.elm位置。如果组件根节点被替换，遍历更新父节点element。然后移除旧节点。
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // 这里是创建新节点并挂载
        createElm(
          vnode,
          insertedVnodeQueue,
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // 递归更新父占位符节点
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

        // 删除旧节点
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }
    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
}
```

### patchVnode

```js
function patchVnode ( oldVnode, vnode, insertedVnodeQueue, ownerArray, index, removeOnly ) {

    if (oldVnode === vnode) {
      return
    }

    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // clone reused vnode
      vnode = ownerArray[index] = cloneVNode(vnode)
    }
    // 复用老节点
    const elm = vnode.elm = oldVnode.elm

    if (isTrue(oldVnode.isAsyncPlaceholder)) {
      if (isDef(vnode.asyncFactory.resolved)) {
        hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
      } else {
        vnode.isAsyncPlaceholder = true
      }
      return
    }

    // reuse element for static trees.
    // note we only do this if the vnode is cloned -
    // if the new node is not cloned it means the render functions have been
    // reset by the hot-reload-api and we need to do a proper re-render.
    if (isTrue(vnode.isStatic) &&
      isTrue(oldVnode.isStatic) &&
      vnode.key === oldVnode.key &&
      (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
      vnode.componentInstance = oldVnode.componentInstance
      return
    }

    // 如果是组件调用组件的prepatch方法
    let i
    const data = vnode.data
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    //开始检查节点下的儿子节点
    const oldCh = oldVnode.children
    const ch = vnode.children
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    if (isUndef(vnode.text)) {
      if (isDef(oldCh) && isDef(ch)) {  // 如果新老节点都有孩子节点,把孩子节点传入.
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) { // 如果新虚拟节点有孩子节点 而 老节点没孩子节点
        if (process.env.NODE_ENV !== 'production') {
          checkDuplicateKeys(ch) // 检查是否有重复key,有就报错
        }
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '') // 检查老节点是否有text,有就直接置为空
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue) //把新节点的孩子节点全加到末尾.
      } else if (isDef(oldCh)) { // 老的有儿子,新的没有
        removeVnodes(oldCh, 0, oldCh.length - 1) //老的儿子全删掉
      } else if (isDef(oldVnode.text)) { // 老的新的都没儿子,老节点有text,删掉
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) { // 如果是字符串节点,且内容不同,用新的
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) { // 如果是组件,且具有postpatch方法,更新组件props ,listener 等属性
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
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

# Vue所有面试题

人有时候就需要被洗脑，被彻底洗脑。

当你真正决定的时候没什么是不可能的。

半山腰太挤了，要去山顶看看。

悟以往之不谏，知来者之可追。

让你快乐的都不能使你强大。

为了长期的欲望，放弃暂时的欲望。

累吗，累就对了。

## 面试题本体

下面是我收集并精选的所有面试题

### 1. 说一下MVVM  
**MVVM**
MVVM是Model-View-ViewModel缩写，也就是把MVC中的Controller演变成ViewModel。
Model层代表数据模型，View代表UI组件，ViewModel是View和Model层的桥梁。
数据数据变化会通知viewModel层自动将数据渲染到页面中，视图变化的时候会通知viewModel层更新数据。

### 2. Vue2.x响应式原理
**响应式**
回答一:
Vue在初始化数据时，会使用Object.defineProperty重新定义data中的所有属性，
当页面使用对应属性时，首先会进行依赖收集，如果属性发生变化会通知相关依赖进行更新操作。

回答二:
数组和对象类型当值变化时如何劫持到。
对象内部通过defineReactive方法，使用Object.defineProperty将属性进行劫持（只会劫持已经存在的属性），
数组则是通过重写数组方法来实现。
这里在回答时可以带出一些相关知识点（比如多层对象是通过递归来实现劫持，顺带提出Vue3中是使用proxy来实现响应式数据）
内部依赖收集是怎样做到的，每个属性都拥有自己的dep属性，存放他所依赖的watcher，当属性变化后会通知自己对应的watcher去更新 （其实后面会讲到每个对象类型自己本身也拥有一个dep属性，这个在$set面试题中在进行讲解）

这里可以引出性能优化相关的内容 （1）对象层级过深，性能就会差 （2）不需要响应数据的内容不要放到data中 （3） Object.freeze() 可以冻结数据

### 3. Vue3.x响应式数据原理
**响应式**
Vue3.x改用Proxy替代Object.defineProperty。因为Proxy可以直接监听对象和数组的变化，并且有多达13种拦截方法。并且作为新标准将受到浏览器厂商重点持续的性能优化。
Proxy只会代理对象的第一层，那么Vue3又是怎样处理这个问题的呢？
判断当前Reflect.get的返回值是否为Object，如果是则再通过reactive方法做代理， 这样就实现了深度观测。
监测数组的时候可能触发多次get/set，那么如何防止触发多次呢？
我们可以判断key是否为当前被代理对象target自身属性，也可以判断旧值与新值是否相等，只有满足以上两个条件之一时，才有可能执行trigger。

### 4. vue2.x中如何监测数组变化
**数组**
使用了函数劫持的方式，重写了数组的方法，Vue将data中的数组进行了原型链重写，指向了自己定义的数组原型方法。
这样当调用数组api时，可以通知依赖更新。
如果数组中包含着引用类型，会对数组中的引用类型再次递归遍历进行监控。
这样就实现了监测数组变化。

**数组**
数组考虑性能原因没有用defineProperty对数组的每一项进行拦截，而是选择重写数组（push,shift,pop,splice,unshift,sort,reverse）方法进行重写。
**为什么数组每一项defineProperty有性能问题**
在Vue中修改数组的索引和长度是无法监控到的。需要通过以上7种变异方法修改数组才会触发数组对应的watcher进行更新。数组中如果是对象数据类型也会进行递归劫持。

那如果想更改索引更新数据怎么办？可以通过Vue.$set()来进行处理 =》 核心内部用的是splice方法

### 5. nextTick实现原理
**nextTick**
在下次 DOM 更新循环结束之后执行延迟回调。nextTick主要使用了宏任务和微任务。根据执行环境分别尝试采用
Promise
MutationObserver
setImmediate
如果以上都不行则采用setTimeout
定义了一个异步方法，多次调用nextTick会将方法存入队列中，通过这个异步方法清空当前队列。

**nextTick**
nextTick中的回调是在下次 DOM 更新循环结束之后执行的延迟回调。在修改数据之后立即使用这个方法，获取更新后的 DOM。原理就是异步方法(promise,mutationObserver,setImmediate,setTimeout)经常与事件环一起来问(宏任务和微任务)

vue多次更新数据，最终会进行批处理更新。内部调用的就是nextTick实现了延迟更新，用户自定义的nextTick中的回调会被延迟到更新完成后调用，从而可以获取更新后的DOM

###  6. Vue生命周期方法有哪些
**生命周期**
beforeCreate是new Vue()之后触发的第一个钩子，在当前阶段data、methods、computed以及watch上的数据和方法都不能被访问。
created在实例创建完成后发生，当前阶段已经完成了数据观测，也就是可以使用数据，更改数据，在这里更改数据不会触发updated函数。可以做一些初始数据的获取，在当前阶段无法与Dom进行交互，如果非要想，可以通过vm.$nextTick来访问Dom。
beforeMount发生在挂载之前，在这之前template模板已导入渲染函数编译。而当前阶段虚拟Dom已经创建完成，即将开始渲染。在此时也可以对数据进行更改，不会触发updated。
mounted在挂载完成后发生，在当前阶段，真实的Dom挂载完毕，数据完成双向绑定，可以访问到Dom节点，使用$refs属性对Dom进行操作。
beforeUpdate发生在更新之前，也就是响应式数据发生更新，虚拟dom重新渲染之前被触发，你可以在当前阶段进行更改数据，不会造成重渲染。
updated发生在更新完成之后，当前阶段组件Dom已完成更新。要注意的是避免在此期间更改数据，因为这可能会导致无限循环的更新。
beforeDestroy发生在实例销毁之前，在当前阶段实例完全可以被使用，我们可以在这时进行善后收尾工作，比如清除计时器。
destroyed发生在实例销毁之后，这个时候只剩下了dom空壳。组件已被拆解，数据绑定被卸除，监听被移出，子实例也统统被销毁。

**生命周期**
生命周期钩子是如何实现的
Vue的生命周期钩子就是回调函数而已，当创建组件实例的过程中会调用对应的钩子方法


**生命周期**
内部主要是使用callHook方法来调用对应的方法。核心是一个发布订阅模式，将钩子订阅好（内部采用数组的方式存储），在对应的阶段进行发布

beforeCreate 在实例初始化之后，数据观测(data observer) 和 event/watcher 事件配置之前被调用。

created 实例已经创建完成之后被调用。在这一步，实例已完成以下的配置：数据观测(data observer)，属性和方法的运算， watch/event 事件回调。这里没有$el

beforeMount 在挂载开始之前被调用：相关的 render 函数首次被调用。

mounted el 被新创建的 vm.$el 替换，并挂载到实例上去之后调用该钩子。

beforeUpdate 数据更新时调用，发生在虚拟 DOM 重新渲染和打补丁之前。

updated 由于数据更改导致的虚拟 DOM 重新渲染和打补丁，在这之后会调用该钩子。

beforeDestroy 实例销毁之前调用。在这一步，实例仍然完全可用。

destroyed Vue 实例销毁后调用。调用后，Vue 实例指示的所有东西都会解绑定，所有的事件监听器会被移

除，所有的子实例也会被销毁。 该钩子在服务器端渲染期间不被调用。

#补充回答:
created 实例已经创建完成，因为它是最早触发的原因可以进行一些数据，资源的请求。(服务端渲染支持created方法)

mounted 实例已经挂载完成，可以进行一些DOM操作

beforeUpdate 可以在这个钩子中进一步地更改状态，这不会触发附加的重渲染过程。

updated 可以执行依赖于 DOM 的操作。然而在大多数情况下，你应该避免在此期间更改状态，因为这可能会导致更新无限循环。 该钩子在服务器端渲染期间不被调用。

destroyed 可以执行一些优化操作,清空定时器，解除绑定事件

### 7. 你的接口请求一般放在哪个生命周期中？
**生命周期**
接口请求一般放在mounted中，但需要注意的是服务端渲染时不支持mounted，需要放到created中。

### 8. 再说一下Computed和Watch
**Computed** **Watch**
Computed本质是一个具备缓存的watcher，依赖的属性发生变化就会更新视图。
适用于计算比较消耗性能的计算场景。当表达式过于复杂时，在模板中放入过多逻辑会让模板难以维护，可以将复杂的逻辑放入计算属性中处理。
Watch没有缓存性，更多的是观察的作用，可以监听某些数据执行回调。当我们需要深度监听对象中的属性时，可以打开deep：true选项，这样便会对对象中的每一项进行监听。这样会带来性能问题，优化的话可以使用字符串形式监听，如果没有写到组件中，不要忘记使用unWatch手动注销哦。

### 9. v-if和v-show的区别
**v-if/v-show**
当条件不成立时，v-if不会渲染DOM元素，v-show操作的是样式(display)，切换当前DOM的显示和隐藏。

**v-if/v-show**
v-if在编译过程中会被转化成三元表达式,条件不满足时不渲染此节点。v-show会被编译成指令，条件不满足时控制样式将对应节点隐藏 （内部其他指令依旧会继续执行）
频繁控制显示隐藏尽量不使用v-if，v-if和v-for尽量不要连用

### 10. 组件中的data为什么是一个函数？
**基础**
一个组件被复用多次的话，也就会创建多个实例。本质上，这些实例用的都是同一个构造函数。如果data是对象的话，对象属于引用类型，会影响到所有的实例。所以为了保证组件不同的实例之间data不冲突，data必须是一个函数。

**基础**
每次使用组件时都会对组件进行实例化操作，并且调用data函数返回一个对象作为组件的数据源。这样可以保证多个组件间数据互不影响

### 11. v-model的原理
**v-model**
v-model本质就是一个语法糖，可以看成是value + input方法的语法糖。 可以通过model属性的prop和event属性来进行自定义。原生的v-model，会根据标签的不同生成不同的事件和属性。

### 12. Vue事件绑定原理说一下
**其他**
原生事件绑定是通过addEventListener绑定给真实元素的，组件事件绑定是通过Vue自定义的$on实现的。

### 13. Vue模板编译原理
**模板**
简单说，Vue的编译过程就是将template转化为render函数的过程。会经历以下阶段：

生成AST树优化codegen
首先解析模版，生成AST语法树(一种用JavaScript对象的形式来描述整个模板)。
使用大量的正则表达式对模板进行解析，遇到标签、文本的时候都会执行对应的钩子进行相关处理。
Vue的数据是响应式的，但其实模板中并不是所有的数据都是响应式的。有一些数据首次渲染后就不会再变化，对应的DOM也不会变化。那么优化过程就是深度遍历AST树，按照相关条件对树节点进行标记。这些被标记的节点(静态节点)我们就可以跳过对它们的比对，对运行时的模板起到很大的优化作用。
编译的最后一步是将优化后的AST树转换为可执行的代码。

**模板**

如何将template转换成render函数(这里要注意的是我们在开发时尽量不要使用template，因为将template转化成render方法需要在运行时进行编译操作会有性能损耗，同时引用带有compiler包的vue体积也会变大。默认.vue文件中的template处理是通过vue-loader来进行处理的并不是通过运行时的编译 - 后面我们会说到默认vue项目中引入的vue.js是不带有compiler模块的)。

1.将template模板转换成ast语法树 - parserHTML

2.对静态语法做静态标记 - markUp

3.重新生成代码 -codeGen

模板引擎的实现原理就是new Function + with来进行实现的

vue-loader中处理template属性主要靠的是vue-template-compiler模块

### 14. Vue2.x和Vue3.x渲染器的diff算法分别说一下
**diff**
简单来说，diff算法有以下过程
同级比较，再比较子节点先判断一方有子节点一方没有子节点的情况(如果新的children没有子节点，将旧的子节点移除)比较都有子节点的情况(核心diff)递归比较子节点
正常Diff两个树的时间复杂度是O(n^3)，但实际情况下我们很少会进行跨层级的移动DOM，所以Vue将Diff进行了优化，从O(n^3) -> O(n)，只有当新旧children都为多个子节点时才需要用核心的Diff算法进行同层级比较。
Vue2的核心Diff算法采用了双端比较的算法，同时从新旧children的两端开始进行比较，借助key值找到可复用的节点，再进行相关操作。相比React的Diff算法，同样情况下可以减少移动节点次数，减少不必要的性能损耗，更加的优雅。
Vue3.x借鉴了
ivi算法和 inferno算法
在创建VNode时就确定其类型，以及在mount/patch的过程中采用位运算来判断一个VNode的类型，在这个基础之上再配合核心的Diff算法，使得性能上较Vue2.x有了提升。(实际的实现可以结合Vue3.x源码看。)
该算法中还运用了动态规划的思想求解最长递归子序列。
(看到这你还会发现，框架内无处不蕴藏着数据结构和算法的魅力)

**diff**
Vue的diff算法是平级比较，不考虑跨级比较的情况。内部采用深度递归的方式 + 双指针的方式进行比较
1.先比较是否是相同节点
2.相同节点比较属性,并复用老节点
3.比较儿子节点，考虑老节点和新节点儿子的情况
4.优化比较：头头、尾尾、头尾、尾头
5.比对查找进行复用
Vue3中采用最长递增子序列实现diff算法

### 15. 再说一下虚拟Dom以及key属性的作用
**虚拟dom**
由于在浏览器中操作DOM是很昂贵的。频繁的操作DOM，会产生一定的性能问题。这就是虚拟Dom的产生原因。
Vue2的Virtual DOM借鉴了开源库snabbdom的实现。
Virtual DOM本质就是用一个原生的JS对象去描述一个DOM节点。是对真实DOM的一层抽象。(也就是源码中的VNode类，它定义在src/core/vdom/vnode.js中。)
VirtualDOM映射到真实DOM要经历VNode的create、diff、patch等阶段。
「key的作用是尽可能的复用 DOM 元素。」
新旧 children 中的节点只有顺序是不同的时候，最佳的操作应该是通过移动元素的位置来达到更新的目的。
需要在新旧 children 的节点中保存映射关系，以便能够在旧 children 的节点中找到可复用的节点。key也就是children中节点的唯一标识。

**虚拟DOM**
Virtual DOM就是用js对象来描述真实DOM，是对真实DOM的抽象，由于直接操作DOM性能低但是js层的操作效率高，可以将DOM操作转化成对象操作，最终通过diff算法比对差异进行更新DOM（减少了对真实DOM的操作）。虚拟DOM不依赖真实平台环境从而也可以实现跨平台。

虚拟DOM的实现就是普通对象包含tag、data、children等属性对真实节点的描述。（本质上就是在JS和DOM之间的一个缓存）

### 16. keep-alive了解吗
**keep-alive**
keep-alive可以实现组件缓存，当组件切换时不会对当前组件进行卸载。
常用的两个属性include/exclude，允许组件有条件的进行缓存。
两个生命周期activated/deactivated，用来得知当前组件是否处于活跃状态。
keep-alive的中还运用了LRU(Least Recently Used)算法。

### 17. Vue中组件生命周期调用顺序说一下
**生命周期**
组件的调用顺序都是先父后子,渲染完成的顺序是先子后父。
组件的销毁操作是先父后子，销毁完成的顺序是先子后父。

加载渲染过程
父beforeCreate->父created->父beforeMount->子beforeCreate->子created->子beforeMount- >子mounted->父mounted

子组件更新过程
父beforeUpdate->子beforeUpdate->子updated->父updated

父组件更新过程
父 beforeUpdate -> 父 updated

销毁过程
父beforeDestroy->子beforeDestroy->子destroyed->父destroyed

### 18.Vue2.x组件通信有哪些方式？
**其他**
父子组件通信
父->子props，子->父 $on、$emit
获取父子组件实例 $parent、$children
Ref 获取实例的方式调用组件的属性或者方法
Provide、inject 官方不推荐使用，但是写组件库时很常用
兄弟组件通信
Event Bus 实现跨组件通信 Vue.prototype.$bus = new Vue
Vuex
跨级组件通信
Vuex
$attrs、$listeners
Provide、inject

**其他**
props和$emit 父组件向子组件传递数据是通过prop传递的，子组件传递数据给父组件是通过$emit触发事件来做到的

$parent,$children 获取当前组件的父组件和当前组件的子组件

$attrs和$listeners A->B->C。Vue 2.4 开始提供了$attrs和$listeners来解决这个问题

父组件中通过provide来提供变量，然后在子组件中通过inject来注入变量。

$refs 获取实例

envetBus 平级组件数据传递 这种情况下可以使用中央事件总线的方式

vuex状态管理

(1) props实现:src/core/vdom/create-component.js:101、 src/core/instance/init.js:74、scr/core/instance/state:64

(2) 事件机制实现: src/core/vdom/create-component.js:101、 src/core/instance/init.js:74、src/core/instance/events.js:12

(3) parent&children实现:src/core/vdom/create-component.js:47、src/core/instance/lifecycle.js:32

(4)provide&inject实现: src/core/instance/inject.js:7

(5)$attrs&$listener实现: src/core/instance/render.js:49、src/core/instance/lifecycle.js:215

(6)$refs实现:src/core/vdom/modules/reg.js:20


### 19. SSR了解吗？
**SSR**
SSR也就是服务端渲染，也就是将Vue在客户端把标签渲染成HTML的工作放在服务端完成，然后再把html直接返回给客户端。
SSR有着更好的SEO、并且首屏加载速度更快等优点。不过它也有一些缺点，比如我们的开发条件会受到限制，服务器端渲染只支持beforeCreate和created两个钩子，当我们需要一些外部扩展库时需要特殊处理，服务端渲染应用程序也需要处于Node.js的运行环境。还有就是服务器会有更大的负载需求。

### 20. 你都做过哪些Vue的性能优化？
**性能优化**
编码阶段

尽量减少data中的数据，data中的数据都会增加getter和setter，会收集对应的watcherv-if和v-for不能连用如果需要使用v-for给每项元素绑定事件时使用事件代理SPA 页面采用keep-alive缓存组件在更多的情况下，使用v-if替代v-showkey保证唯一使用路由懒加载、异步组件防抖、节流第三方模块按需导入长列表滚动到可视区域动态加载图片懒加载
SEO优化

预渲染服务端渲染SSR
打包优化

压缩代码Tree Shaking/Scope Hoisting使用cdn加载第三方模块多线程打包happypacksplitChunks抽离公共文件sourceMap优化
用户体验

骨架屏PWA
还可以使用缓存(客户端缓存、服务端缓存)优化、服务端开启gzip压缩等。

### 21. hash路由和history路由实现原理说一下
**路由**
location.hash的值实际就是URL中#后面的东西。
history实际采用了HTML5中提供的API来实现，主要有history.pushState()和history.replaceState()。


### 22. Vue.mixin的使用场景和原理
**mixin**
ue.mixin的作用就是抽离公共的业务逻辑，原理类似“对象的继承”，当组件初始化时会调用mergeOptions方法进行合并，
采用策略模式针对不同的属性进行合并。如果混入的数据和本身组件中的数据冲突，会采用“就近原则”以组件的数据为准。

mixin中有很多缺陷 "命名冲突问题"、"依赖问题"、"数据来源问题",这里强调一下mixin的数据是不会被共享的

### 23. Vue.set方法是如何实现的?
为什么$set可以触发更新,我们给对象和数组本身都增加了dep属性。当给对象新增不存在的属性则触发对象依赖的watcher去更新，当修改数组索引时我们调用数组本身的splice方法去更新数组

### 24.$attrs是为了解决什么问题出现的？应用场景有哪些？provide/inject 不能解决它能解决的问题吗？
$attrs主要的作用就是实现批量传递数据。provide/inject更适合应用在插件中，主要是实现跨级数据传递

### 25.Vue的组件渲染流程
①在渲染父组件时会创建父组件的虚拟节点,其中可能包含子组件的标签
②在创建虚拟节点时,获取组件的定义使用Vue.extend生成组件的构造函数。 
③将虚拟节点转化成真实节点时，会创建组件的实例并且调用组件的$mount方法。 
④所以组件的创建过程是先父后子

### 26. Vue.use是干什么的?原理是什么?
Vue.use是用来使用插件的，我们可以在插件中扩展全局组件、指令、原型方法等。

### 27.vue-router有几种钩子函数?具体是什么及执行流程是怎样的?
路由钩子的执行流程, 钩子函数种类有:全局守卫、路由守卫、组件守卫

#完整的导航解析流程:
①导航被触发。
②在失活的组件里调用 beforeRouteLeave 守卫。
③调用全局的 beforeEach 守卫。
④在重用的组件里调用 beforeRouteUpdate 守卫 (2.2+)。
⑤在路由配置里调用 beforeEnter。
⑥解析异步路由组件。
⑦在被激活的组件里调用 beforeRouteEnter。
⑧调用全局的 beforeResolve 守卫 (2.5+)。
⑨导航被确认。
⑩调用全局的 afterEach 钩子。
⑪触发 DOM 更新。
⑫调用 beforeRouteEnter 守卫中传给 next 的回调函数，创建好的组件实例会作为回调函数的参数传入


### 28.vue-router两种模式的区别？
#核心答案:
hash模式、history模式

hash模式：hash + hashChange 兼容性好但是不美观
history模式 : historyApi+popState 虽然美观，但是刷新会出现404需要后端进行配置



### 29. 函数式组件的优势及原理
函数式组件的特性,无状态、无生命周期、无this


### 30.v-if与v-for的优先级
v-for和v-if不要在同一个标签中使用,因为解析时先解析v-for在解析v-if。如果遇到需要同时使用时可以考虑写成计算属性的方式。

### 31.组件中写name选项又哪些好处及作用
可以通过名字找到对应的组件 (递归组件)
可用通过name属性实现缓存功能 (keep-alive)
可以通过name来识别组件 (跨级组件通信时非常重要)

### 32.Vue事件修饰符有哪些？其实现原理是什么?
事件修饰符有：.capture、.once、.passive 、.stop、.self、.prevent、

### 33.Vue.directive源码实现?
把定义的内容进行格式化挂载到Vue.options属性上

### 34.如何理解自定义指令?
指令的实现原理，可以从编译原理=>代码生成=>指令钩子实现进行概述

1.在生成ast语法树时，遇到指令会给当前元素添加directives属性

2.通过genDirectives生成指令代码

3.在patch前将指令的钩子提取到cbs中,在patch过程中调用对应的钩子

4.当执行指令对应钩子函数时，调用对应指令定义的方法

### 35. 谈一下你对vuex的个人理解
vuex是专门为vue提供的全局状态管理系统，用于多个组件中数据共享、数据缓存等。（无法持久化、内部核心原理是通过创造一个全局实例 new Vue）

衍生的问题action和mutation的区别
核心方法: replaceState、subscribe、registerModule、namespace(modules)

### 36.Vue中slot是如何实现的？什么时候用它?
普通插槽（模板传入到组件中，数据采用父组件数据）和作用域插槽（在父组件中访问子组件数据）

### 37.keep-alive平时在哪使用？原理是?
keep-alive主要是缓存，采用的是LRU算法。 最近最久未使用法







## 参考文章

[童欧巴20+Vue面试题](https://juejin.im/post/6844904084374290446)

珠峰课程Vue阶段面试题
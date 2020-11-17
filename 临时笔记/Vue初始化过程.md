
# Vue 初始化过程

以文件引用顺序作为路线。

## 外层处理

引用vue并逐层给vue挂载各种全局方法

### 1.进入 plateforms 文件夹

 web 和 weex 个文件,是平台化差异的解决包,哪个平台用哪个。

### 2.进入 plateforms/web/entry-runtime-with-compiler 文件

浏览器直接使用vue时入口是这里，可在运行时解析生成render函数,包含引用了entry-runtime

这个文件里重写了$mount,在$mount之前,干了两件事

1.判断是否有render函数,没有则判断是否有template,再没有判断获取el,都没有就创建空div,最后返回template

2.传入template,调用compileToFunctions,返回render函数。

之后调用原版的$mount

### 3.进入 plateforms/web/runtime/index 文件

这里干了三件事

1.扩展了全局指令和全局组件如 'v-show' ,'<transition>' 供用户调用

2.挂载全局方法$mount,__patch__ 供用户调用

3.推广提示了下载vue的官方浏览器调试插件.......

### 4.进入 core/index 文件

调用初始化全局api函数 initGlobalAPI()

包括：
    vue.config  
    内部的工具方法 warn,extend，mergeOptions，defineReactive。 
    vue.set vue.delete vue.nextTick
    vue.boservable
    定义好默认属性 Vue.options.components/.directives/.filters = {} 
    keepAlive组件 mergeOptions
    .use() .extend() .mixin()

## Vue核心

核心的构造函数

### 1. 进入 instance/index


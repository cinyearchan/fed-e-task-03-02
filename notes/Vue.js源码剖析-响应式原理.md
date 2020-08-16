# Vue 源码解析-响应式原理

## 要点

- Vue.js 的静态成员和实例成员初始化过程
- 首次渲染的过程
- **数据响应式原理**



## 调试设置

### 打包

- 打包工具 Rollup

  - Vue.js 源码的打包工具使用的是 Rollup，比 Webpack 轻量
  - Webpack 把所有文件当做模块，Rollup 只处理 js 文件更适合处理 Vue.js 这样的库
  - Rollup 打包不会生成冗余的代码

- 安装依赖

  ```shell
  npm i
  ```

- 设置 sourcemap

  - package.json 文件中的 dev 脚本中添加参数 --sourcemap

    ```js
    "dev": "rollup -w -c scripts/config.js --sourcemap --environment TARGET:web-full-dev"
    ```

- 执行 dev

  - `npm run dev` 执行1打包，用的是 rollup，-w 参数是监听文件的变化，文件变化自动重新打包



## Vue 的不同构建版本

- `npm run build` 重新打包所有文件
- 官方文档-对不同构建版本的解释
- dist/README.md

| -                        | UMD                | CommonJS              | ES Module          |
| ------------------------ | ------------------ | --------------------- | ------------------ |
| Full                     | vue.js             | vue.common.js         | vue.esm.js         |
| Runtime-only             | vue.runtime.js     | vue.runtime.common.js | vue.runtime.esm.js |
| Full(production)         | vue.min.js         |                       |                    |
| Runtime-only(production) | vue.runtime.min.js |                       |                    |

术语

- 完整版：同时包含**编译器**和**运行时**的版本
- 编译器：用来将模板字符串编译成为 JavaScript 渲染函数的代码，体积大、效率低
- 运行时：用来创建 Vue 实例、渲染并处理虚拟 DOM 等的代码，体积小、效率高。基本上就是除去编译器的代码
- UMD：通用的模块版本，支持多种模块方式。vue.js 默认文件就是运行时+编译器的 UMD 版本
- CommonJS(cjs)：CommonJS 版本用来配合老的打包工具比如 Browserify 或 webpack1
- ES Module：从 2.6 开始 Vue 会提供两个 ES Modules(ESM) 构建文件，为现代打包工具提供的版本
  - ESM 格式被设计为可以被静态分析，所以打包工具可以利用这一点来进行 tree-shaking 并将用不到的代码排除出最终的包
  - ES6 模块与 CommonJS 模块的差异



## 寻找入口文件

- 查看 dist/vue.js 的构建过程

### 执行构建

```sh
npm run dev
# "dev": "rollup -w -c scripts/config.js --sourcemap --environment TARGET:web-full-dev"
# --environment TARGET:web-full-dev 设置环境变量 TARGET
```

- script/config.js 的执行过程
  - 作用： 生成 rollup 构建的配置文件
  - 使用环境变量 TARGET = web-full-dev

```js
// 判断环境变量是否有 TARGET
// 如果有的话，使用 genConfig() 生成 rollup 配置文件
if (process.env.TARGET) {
  module.exports = genConfig(process.env.TARGET)
} else {
  // 否则获取全部配置
  exports.getBuild = genConfig
  exports.getAllBuilds = () => Object.keys(builds).map(genConfig)
}
```



## 从入口开始

`src/platform/web/entry-runtime-with-compiler.js`

```js
// 保留 Vue 实例的 $mount 方法
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Elment,
  // 非 ssr 情况下为 false，ssr 时为 true
  hydrating?: boolean
): Component {
  // 获取 el 对象
  
  /* istanbul ignore if */
  // el 不能是 body 或者 html
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
    	`Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }
  
  const options = this.$options
  // resolve template/el and convert to render function
  // 把 template/el 转换成 render 函数
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        // 如果模板是 id 选择器
        if (template.charAt(0) === '#') {
          // 获取对应的 DOM 对象的 innerHTML
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
            	`Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        // 如果模板是元素，返回元素的 innerHTML
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
      }
    }
  }
  // 调用 mount 方法，渲染 DOM
  return mount.call(this, el, hydrating)
}
```

- 阅读源码记录
  - el 不能是 body 或者 html 标签
  - 如果没有 render，把 template 转换成 render 函数
  - 如果要 render 方法，直接调用 render 挂载 DOM

### 通过查看源码解决以下问题

观察以下代码，阅读源码后，下方代码会在页面上输出什么

```js
const vm = new Vue({
  el: '#app',
  template: '<h3>Hello template</h3>',
  render (h) {
    return h('h4', 'Hello render')
  }
})
```

会直接输出 `<h4>Hello render</h4>`

调用堆栈

| Vue.$mount  entry-runtime-with-compiler.js |
| ------------------------------------------ |
| Vue._init  init.js                         |
| Vue  index.js                              |



## Vue 初始化的过程

四个导出 Vue 的模块

1. src/platforms/web/entry-runtime-with-compiler.js
   - web 平台相关的入口
   - 重写了平台相关的 $mount() 方法
   - 注册了 Vue.compile() 方法，传递一个 HTML 字符串返回 render 函数
2. src/platforms/web/runtime/index.js
   - web 平台相关
   - 注册和平台相关的全局指令：v-model、v-show
   - 注册和平台相关的全局组件：v-transition、v-transition-group
   - 全局方法
     - `__patch__`：把虚拟 DOM 转换成真实 DOM
     - `$mount`：挂载方法
3. src/core/index.js
   - 与平台无关
   - 设置了 Vue 的静态方法，initGlobalAPI(Vue)
4. src/core/instance/index.js
   - 与平台无关
   - 定义了构造函数，调用了 this._init(options) 方法
   - 给 Vue 中混入了常用的实例成员



## Vue 初始化——静态成员

src/core/global-api/index.js

```js
export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  // 初始化 Vue.config 对象
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  // 这些工具方法不视作全局 API 的一部分，除非你已经意识到某些风险，否则不要去依赖它们
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }
  // 静态方法 set/delete/nextTick
  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  // 2.6 explicit observable API
  // 让一个对象可响应
  Vue.observable = <T>(obj: T): T => {
    observe(obj)
    return obj
  }
  // 初始化 Vue.options 对象，并给其扩展
  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
      Vue.options[type + 's'] = Object.create(null)
    })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue
  // 设置 keep-alive 组件
  extend(Vue.options.components, builtInComponents)
  // 注册 Vue.use() 用来注册插件
  initUse(Vue)
  // 注册 Vue.mixin() 实现混入
  initMixin(Vue)
  // 注册 Vue.extend() 基于传入的 options 返回一个组件的构造函数
  initExtend(Vue)
  // 注册 Vue.directive()、Vue.component()、Vue.filter()
  initAssetRegisters(Vue)
}
```



## Vue 初始化——实例成员

src/core/instance/index.js

```js
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'
// 此处不用 class 的原因是为了方便后续给 Vue 实例混入实例成员
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  // 调用 _init() 方法
  this._init(options)
}
// 注册 vm 的 _init() 方法，初始化 vm
initMixin(Vue)
// 注册 vm 的 $data/$props/$set/$delete/$watch
stateMixin(Vue)
// 初始化事件相关方法
// $on/$once/$off/$emit
eventsMixin(Vue)
// 初始化生命周期相关的混入方法
// _update/$forceUpdate/$destory
lifecycleMixin(Vue)
// 混入 render 
// $nextTick/_render
renderMixin(Vue)

export default Vue
```



## Vue 首次渲染过程

1. Vue 初始化，实例成员，静态成员
2. new Vue()
3. this._init()
4. vm.$mount()
   - src/platforms/web/entry-runtime-with-compiler.js
   - 如果没有传递 render，把模板编译成 render 函数
   - compileToFunctions() 生成 render() 渲染函数
   - options.render = render
5. vm.$mount()
   - src/platforms/web/runtime/index.js
   - mountComponent()
6. mountComponent(this, el)
   - src/core/instance/lifecycle.js
   - 判断是否有 render 选项，如果没有但是传入了模板，并且当前是开发环境的话会发送警告
   - 触发 beforeMount
   - 定义 updateComponent
     - `vm._update(vm._render(), ...)`
     - `vm._render()` 渲染，渲染虚拟 DOM
     - vm.update() 更新，将虚拟 DOM 转换成真实 DOM
   - 创建 Watcher 实例
     - updateComponent 传递
     - 调用 get() 方法
   - 触发 mounted
   - return vm
7. watcher.get()
   - 创建完 watcher 会调用一次
   - 调用 updateComponent()
   - 调用 `vm._render()` 创建 VNode
     - 调用 render.call(vm._renderProxy, vm.$createElement)
     - 调用实例化时 Vue 传入的 render()
     - 或者编译 template 生成的 render()
     - 返回 VNode
   - 调用 vm._update(vnode, ...)
     - 调用 `vm.__patch__(vm.$el, vnode)` 挂载真实 DOM
     - 记录 vm.$el



## 数据响应式原理

### 通过查看源码解决下面问题

- `vm.msg = { count: 0 }`，重新给属性赋值，是否是响应式的
- `vm.arr[0] = 4`，给数组元素赋值，视图是否会更新
- `vm.arr.length = 0`，修改数组的length，视图是否会更新
- `vm.arr.push(4)`，视图是否会更新

### 响应式处理的入口

整个响应式处理的过程是比较复杂的

- src/core/instance/init.js

  - `initState(vm)` vm 状态的初始化
  - 初始化了 `_data`、`_props`、`methods` 等

- src/core/instance/state.js

  ```js
  // 数据的初始化
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  ```

  

### Watcher 类

- watcher 分为三种，Computed Watcher、用户 Watcher（侦听器）、渲染 Watcher

- 渲染 Watcher 的创建时机

  - src/core/instance/lifecycle.js

    ```typescript
    export function mountComponent (
    	vm: Components,
      el: ?Element,
      hydrating?: boolean
    ): Component {
      vm.$el = el
      ...
      callHook(vm, 'beforeMount')
      
      let updateComponent
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        ...
      } else {
        
      }
    }
    ```

    

### 调试响应式数据执行过程

- 数组响应式处理的核心过程和数组收集依赖的过程
- 当数组的数据改变的时候 watcher 的执行过程





### `vm.$delete`

- 功能

  删除对象的属性。如果对象是响应式的，确保删除能触发更新视图。这个方法主要用于避开 Vue 不能检测到属性被删除的限制，但是你应该很少会使用它

  > 注意：目标对象不能是一个 Vue 实例或 Vue 实例的根数据对象

  示例

  ```js
  vm.$delete(vm.obj, 'msg')
  ```

  定义位置

  - Vue.delete()

    - Global-api/index.js

  ```js
  // 静态方法 set/delete/nextTick
  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick
  ```
  ```
  
  - `vm.$delete()`
    - Instance/index.js
  
  ​```js
  // 注册 vm 的 $data/$props/$set/$delete/$watch
  stateMixin(Vue)
  
  // instance/state.js
  Vue.prototype.$set = set
  Vue.prototype.$delete = del
  ```
```

  

  

### `vm.$watch`  

`vm.$watch(expOrFn, callback, [options])`

-  功能

  观察 Vue 实例变化的一个表达式或计算属性函数。回调函数得到的参数为新值和旧值。表达式只接受监督的键路径，对于更复杂的表达式，用一个函数取代

- 参数

  - expOrFn：要监视的 `$data` 中的属性，可以是表达式或函数
  - callback：数据变化后执行的函数
    - 函数：回调函数
    - 对象：具有 handler 属性（字符串或者函数），如果该属性为字符串则 methods 中相应的定义
  - options：可选的选项
    - deep：布尔类型，深度监听
    - Immediate：布尔类型，是否立即执行一次回调函数

- 示例

  ```js
  const vm = new Vue({
    el: '#app',
    data: {
      a: '1',
      b: '2',
      msg: 'Hello Vue',
      user: {
        firstName: '诸葛',
        lastName: '亮'
      }
    }
  })
  // expOrFn 是表达式
  vm.$watch('user', function (newVal, oldVal) {
    this.user.fullName = newVal.firstName + ' ' + newVal.lastName
  }, {
    immediate: true,
    deep: true
  })
```

### 异步更新队列 nextTick()
- Vue 更新 DOM 是异步执行的、批量的
	- 在下次 DOM 更新循环结束之后执行延迟回调。在修改数据之后立即使用这个方法，获取更新后的 DOM
- `vm.$nextTick(function () {/* 操作 DOM */}) / Vue.nextTick(function() {})`

#### 定义位置

- src/core/instance/render.js

```js
Vue.prototype.$nextTick = function(fn: Function) {
  return nextTick(fn, this)
}
```

#### 源码

- 手动调用 vm.$nextTick()
- 在 Watcher 的 queueWatcher 中执行 nextTick()
- src/core/util/next-tick.js
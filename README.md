# 一、简答题

1. 请简述 Vue 首次渲染的过程

   - `Vue` 初始化，实例成员、静态成员
   - `new Vue()`
   - `this._init()`
   - `vm.$mount()`
     - `src/platforms/web/entry-runtime-with-compiler.js`
     - 如果没有传递 `render`，把模板编译成 `render` 函数
     - `compileToFunctions` 生成 `render()` 渲染函数
     - `options.render = render`
   - `vm.$mount()`
     - `src/platforms/web/runtime/index.js`
     - `mountComponent`
   - `mountComponent(this, el)`
     - `src/core/instance/lifecycle.js`
     - 判断是否有 `render` 选项，如果没有但是传入了模板，并且当前是开发环境的话会发送警告
     - 触发 `beforeMount`
     - 定义 `updateComponent`
       - `vm._update(vm._render(), ...)`
       - `vm._render(` 渲染，渲染虚拟 DOM
       - `vm._update()` 更新，将虚拟 DOM 转换成真实 DOM
     - 创建 `Watcher` 实例
       - `updateComponent` 传递
       - 调用 `get()` 方法
     - 触发 `mounted`
     - `return vm`
   - `watcher.get()`
     - 创建完 `watcher` 会调用一次 `get`
     - 调用 `updateComponent()`
     - 调用 `vm._render()` 创建 VNode
       - 调用 `render.call(vm._renderProxy, vm.$createElement)`
       - 调用实例化时 Vue 传入的 `render()`
       - 或者编译 template 生成的 `render()`
       - 返回 VNode
     - 调用 `vm._update(vnode, ...)`
       - 调用 `vm.__patch__(vm.$el, vnode)` 挂载真实 DOM
       - 记录 `vm.$el`

2. 请简述 Vue 响应式原理

   - `initState() -> initData() -> observe()`
   - `observe(value)`
     - `src/core/observer/index.js`
     - 判断 value 是否是对象，如果不是对象直接返回
     - 判断 value 对象是否有 `__ob__`，如果有直接返回
     - *如果没有*，创建 `observer` 对象
     - 返回 `observer` 对象
   - `Observer`
     - `src/core/observer/index.js`
     - 给 value 对象定义不可枚举的 `__ob__` 属性，记录当前的 `observer` 对象
     - *数组的响应式处理*——拦截数组原型方法
     - *对象的响应式处理*—— `Object.defineProperty()`，调用 `walk` 方法
   - `defineReactive`
     - `src/core/observer/index.js`
     - 为每一个属性创建 `dep` 对象
     - 如果当前属性的值是对象，调用 `observe`
     - *定义 getter*
       - *收集依赖*
       - 返回属性的值
     - *定义 setter*
       - 保存新值
       - 如果新值是对象，调用 `observe`
       - *派发更新（发送通知）*，调用 `dep.notify()`
   - 依赖收集
     - 在 watcher 对象的 `get` 方法中调用 `pushTarget` 记录 `Dep.target` 属性
     - 访问 `data` 中的成员的时候收集依赖，`defineReactive` 的 `getter` 中收集依赖
     - 把属性对应的 `watcher` 对象添加到 `dep` 的 `subs` 数组中
     - 给 `childOb` 收集依赖，目的是子对象添加和删除成员时发送通知
   - `Watcher`
     - `dep.notify()` 在调用 `watcher` 对象的 `update()` 方法
     - `queueWatcher()` 判断 `watcher` 是否被处理，如果没有的话添加到 `queue` 队列中，并调用 `flushSchedulerQueue()`
     - `flushSchedulerQueue()`
       - 触发 `beforeUpdate` 钩子函数
       - 调用 `watcher.run()`
         - `run() -> get() -> getter() -> updateComponent`
       - 清空上一次的依赖
       - 触发 `actived` 钩子函数
       - 触发 `updated` 钩子函数

3. 请简述虚拟 DOM 中 Key 的作用和好处

   - 作用
     - 以便能够跟踪每个节点的身份，在进行比较的时候，会基于 key 的变化重新排列元素顺序，从而重用和重新排序现有元素，并且会移除 key 不存在的元素，方便让 vnode 在 diff 过程中找到对应的节点，成功复用
   - 好处
     - 减少 dom 操作，减少 diff 和渲染所需的时间，提升性能

4. 请简述 Vue 中模板编译的过程

   - `compileToFunctions(template, ...)`

     - 先从缓存中加载编译好的 `render` 函数
     - 缓存中没有调用 `compile(template, options)`
- `compile(template, options)`
   
  - 合并 `options`
     - `baseCompile(template.trim(), finalOptions)`
   - `baseCompile(template.trim(), finalOptions)`

     - `parse()`

       - 把 `template` 转换成 `AST tree`
	 - `optimize()`
          - 标记 `AST tree` 中的静态 `sub trees`
	       - 检测到静态子树，设置为静态，不需要在每次重新渲染的时候重新生成节点
          - `patch` 阶段跳过静态子树
     - `generate()`
       - `AST tree` 生成 js 的创建代码
   - `compileToFunctions(template, ...)`
     - 继续把上一步中生成的字符串形式 js 代码转换成函数
     - `createFunction()`
     - `render` 和 `staticRenderFns` 初始化完毕，挂载到 Vue 实例的 `options` 对应的属性中
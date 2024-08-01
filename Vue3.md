## Vue的设计与实现

### 一、框架设计

> Vue3是一款声明式框架

#### 1.1为什么选择声明式

声明式的选择主要体现在性能与可维护性上的权衡，jquery是典型的命令式框架。

命令式框架和声明式框架在性能上的差异主要体现在修改时，命令式框架可以直观的进行对应位置的修改操作，但是声明式框架要先去寻找要修改的位置，再进行修改，我们把修改的性能消耗定义为A，找出差异的为B

命令式的更新性能消耗为A

声明式的更新性能消耗为A+B

所以声明式的 性能消耗一定是高于命令式的，既然声明式的性能消耗更高为什么还要选择声明式呢？这是因为声明式的代码的可维护性更高，并且极致优化的命令式代码很难书写，并且只要我们让性能消耗B尽可能的减小，那么我们在性能上的差距就可以无限贴近于命令式。

#### 1.2为什么选择运行时+编译时

编译时的架构是指将一段代码直接编译成命令式代码，运行时的架构是指将一段代码通过一个黑盒进行运行，然后再生成命令式代码执行，vue3提供了编译时➕运行时的模式，让我们可以在写类似于html的模版或者直接写树形结构调用Render函数，还是提高了可维护性和书写的便捷性

编译时的优点：因为对代码进行了编译工作，所以可以对用户提供的内容进行分析

### 二、框架设计的核心

#### 2.1提升用户的体验

会用console.warn来反馈出一些方便定位问题所在的信息

小tip：

在我们打印ref数据时，console.log(ref(0))控制台的输出一般是2-1这个样子的

<img src="assets/image-20230905105907956.png" alt="image-20230905105907956" style="zoom:70%;" />

​        	                                              图 2-1

这个显示是不直观的，浏览器支持我们自定义编写formatter从而自定义输出形式，在vue3中可以搜索到initCustomFormatter的函数，该函数就是用于开发环境初始化自定义formatter的，可以在设置中打开

<img src="assets/image-20230905110116019.png" alt="image-20230905110116019" style="zoom:80%;" />

​              	                                              图 2-2

这样我们再打印时可以得到直观的内容

![image-20230905110157708](assets/image-20230905110157708.png)

#### 2.2控制框架代码的体积

##### 2.2.1剔除不必要的调试体验代码

因为要提高用户体验，所以框架在设计的时候会有很多的用于发出警告和检查的代码，但这些代码在生产环境中时不需要的，vue在做这个设计的时候采用了环境变量判断的模式

```js
if (__DEV__ && !res) {
  warn(
  	`Faild to mount app......`
  )
}
```

可以看到\__DEV__环境变量在开发环境下为true，在生产环境下为false，如果为false这段代码就永远不会执行了，在用rollup.js进行打包的时候就会被移除，有效的减少了生产环境下的代码体积

##### 2.2.2剔除不必要的废代码或无意义代码

```js
export const isHTMLTag = /*#__PURE__*/ makeMap(HTML_TAGS)
```

也会用/\*#\__PURE__*/来表示这个函数没有副作用，如果该函数只是读一个结果，并且该结果没有赋值给别的单位，最终也会被删除

##### 2.2.3剔除没用的功能代码

框架在设计时会随着更新的迭代产生一些废弃的api，在我们明确我们项目中没有使用时，可以通过相应的配置进行删除处理，例如vue3中如果我们明确我们使用的是Composition API就可以用\__VUE_OPTIONS_API__开关来关闭选项式API的特性，从而减小代码体积

#### 2.3 错误处理

提供统一的错误处理函数

#### 2.4良好的TS支持

### 三、Vue.js 3的设计思路

#### 3.1 声明式地描述UI

> 编写前端页面要涉及的内容
>
> 1、DOM元素
>
> 2、元素的属性
>
> 3、元素的事件
>
> 4、元素的层级结构

Vue选择了和原生基本一致的描述方式来进行**模版**的书写，当然我们也可以用js对象的方式来描述

##### 3.1.1模版描述ui

```vue
<templete>
  <div class="dom" :id="coustomId" @click="handler">
  	<div />
  </div>
</templete>
```

##### 3.1.2js对象描述ui

```js
const vnode = {
  // 标签名称
  tag: 'div',
  // 标签属性
  props: {
    class: 'dom',
    id: coustomId
    onClick: handler
  },
  // 子节点
  children: [
    { tag: 'div' }
  ]
}
export default {
  render() {
    return vnode
  }
}

```

利用h函数

```js
import { h } from 'vue'
export default {
  render () {
    return h('h1', { onClick:handler })
  }
}
```

#### 3.2初始渲染器

Vue先通过渲染函数(.render ) 拿到虚拟DOM，之后再通过渲染器将虚拟DOM转化为真实DOM，如图3-1所示

<img src="assets/image-20230913143413877.png" alt="image-20230913143413877" style="zoom:50%;" />

​														图3-1

```js
const vnode = {
  // 标签名称
  tag: 'div',
  // 标签属性
  props: {
    class: 'dom',
    onClick: handler
  },
  // 子节点
  children: 'click me'
}
export default {
  render() {
    return title
  }
}		
```

先解释一下上面的代码：

- tag用来描述标签名称，所以div就是一个\<div>标签
- props是一个对象，用来描述\<div>标签的属性、事件等内容
- children用来描述标签的子节点，上面代码中是一个字符串，所以代码\<div>有一个文本节点

接下来我们实现一个渲染器

```js
function renderer (vnode, container) {
  const { tag, props, children} = vnode
  // 生成DOM节点
  const el = document.createElement(tag)
  // 处理props属性
  for ( const key in props) {
    // 如果是on开头说明是事件
    if(/^on/.test(key)) {
      el.addEventListener(key.slice(2).toLowerCase(), props[key])
    } else {
      el.setAttribute(key, props[key])
    }
  }
  // 处理children
  if (typeof children === 'string') el.appendChild(document.createTextNode(children))
  else if (Array.isArray(children)) children.forEach(child => renderer(child, el))
  container.appendChild(el)
}
```

当然这只是首次渲染，渲染器复杂的地方在于更新阶段，例如文本内容的变更从click me变更为 click again，渲染器应该只更新元素的文本内容，而不需要走一遍完整的创建过程

#### 3.3组件的本质

组件的本质就是DOM元素的封装，有点像js中的fragment切片，所以我们可以定义一个函数，函数的返回值就是这个页面要渲染的内容

```js
const MyComponent =function () {
  return {
    tag: 'div',
    props: {
      onClick: () => alert(1)
    },
    children: 'click me'
  }
}
```

重构渲染器renderer函数的tag处理部分

```js
function renderer (vnode, container) {
  // 是dom节点
  if (typeof vnode.tag === 'string') mountElement(vnode, container)
  // 是组件节点
  else if (typeof vnode.tag === 'function') mountComponent(vnode, container)
}
```

**mountElement**是之前的renderer函数

```js
function mountElement (vnode, container) {
  const { tag, props, children} = vnode
  // 生成DOM节点
  const el = document.createElement(tag)
  // 处理props属性
  for ( const key in props) {
    // 如果是on开头说明是事件
    if(/^on/.test(key)) {
      el.addEventListener(key.slice(2).toLowerCase(), props[key])
    } else {
      el.setAttribute(key, props[key])
    }
  }
  // 处理children
  if (typeof children === 'string') el.appendChild(document.createTextNode(children))
  else if (Array.isArray(children)) children.forEach(child => renderer(child, el))
  container.appendChild(el)
}
```

**mountComponent**如下，主要是为了生成虚拟DOM，然后再去调用renderer

```js
function mountComponent (vnode, container) {
  // 获取虚拟DOM
  const sunTree = vnode.tag()
  renderer(sunTree, container)
}
```

当然我们知道的Vue的组件展示形式是一个对象，所以最终**renderer、mountComponent**应该是下方这个样子的

**renderer**如下

```js
function renderer (vnode, container) {
  // 是dom节点
  if (typeof vnode.tag === 'string') mountElement(vnode, container)
  // 是组件节点
  else if (typeof vnode.tag === 'object') mountComponent(vnode, container)
}
```

**mountComponent** 如下

```js
function mountComponent (vnode, container) {
  // 获取虚拟DOM
  const sunTree = vnode.tag.render()
  renderer(sunTree, container)
}	
```

#### 3.4模版的工作原理

模版的工作原理其实主要是借助于编辑器，我们知道在写vue时会导出一个对象，编译器让模版生成作用的原理就是在导出的对象上生成一个render函数，返回vnode，**并在进行分析时，将可能发生变化的数据在vnode中标识出来，方便渲染器进行更新时找到哪个地方可能发生变化**

```js
{
  render () {
    return {
      tag: 'div',
      props: {
        id: 'foo',
        class: cls
      },
      patchFlags: 1 //假设数字1代表class是动态的
    }
  }
}
```

### 四、响应系统的作用与实现

#### 4.1响应式数据和副作用函数

在介绍响应式系统之前，我们要先明确一个概念，副作用函数，什么是副作用函数？，用以下代码为例：

```js
function effect () {
  document.body.innerText = 'hello'
}
```

我们修改了body的值为hello，但是body我们不仅在effect函数中可以获取到，在任何地方都可以获取到，所以我们说effect函数是一个副作用函数，总结：副作用函数就是在函数内修改了非仅函数内部可访问的变量的值，对外部数据产生了影响

那这个和响应式有什么关系呢，我们改造一下上面的代码

```js
const obj = {text: 'hello'}
function effect () {
  document.body.innerText = obj.text
}
obj.text = 'change'
```

代码中我们让body的值等于了obj对象的text属性，之后修改了obj的text属性的值，修改后body的值没有跟随着改变，我们期望的是obj的text的值发生了变化，effect函数要重新执行，body的值也跟随着变化，这就是响应式

#### 4.2响应式数据的基本实现

我们期望数据变化，使用数据了的副作用函数就重新执行，这涉及到了两个问题，就是我们怎么知道哪些副作用绑定了该数据，以及我们如何通过副作用函数重新执行，es2015中的proxy可以帮我们解决对应的问题，proxy允许我们在数据读取和修改时创建钩子函数，这样我们就可以在数据读取也就是应用时，将副作用函数收集起来，等到数据修改时，再将副作用函数提取出来进行执行

按照该思路我们进行实现

```js
// 储存副作用函数的桶
const bucket = new Set()
const obj = new Proxy({text: 'hello'},{
  get(target, key) {
    // 将副作用函数收集起来
    bucket.add(effect)
    return target[key]
  },
  set(target, key, newValue) {
    const setRes = Reflect.set(...arguments)
    // 将副作用函数从桶中拿出来执行
    bucket.forEach(fn=>fn())
    return setRes
  }
})
const effect = ()=>{
  document.body.innerText = obj.text
}
effect()
setTimeout(()=>obj.text = 'change',1000)
```

至此我们就实现了一个基础版的响应式，但是其中还有很多的问题，例如我们在收集副作用的时候明确了副作用函数是effect，我们在后面会解决这些问题

#### 4.3设计一个完善的响应式系统

我们先解决副作用函数的命名问题，我们期望可以明确一个变量指向副作用函数，同时期望使用的人可以随意给自己的副作用函数命名，我们可以提供一个函数来进行对应的转换，代码如下

```js
let activeEffect = undefined
function effect (fn) {
  activeEffect = fn
  fn()
}
```

至此我们已经解决了副作用函数的命名问题，但随之而来的问题是我们的副作用收集并不够明确，obj是一个对象，对象会有很多的属性，我们在收集副作用时，将所有的副作用都与对象绑定了，然而这个并不够准确，例如下面的例子

```js
effect(()=>{
  console.log('执行了副作用函数')
  document.body.innerText = obj.text
})
obj.noProps = true
```

我们会发现，在修改obj的noProps属性时，副作用函数也执行了，然而我们期望的是仅修改副作用函数中用到的变量的对应的属性的时候才会执行对应的副作用函数，因此我们需要修改一下我们的绑定关系，从最开始的对象变为对象的对应属性，我们从新设计桶的数据结构

![image-20240731154243583](/Users/admin/Desktop/yue-vue/assets/image-20240731154243583.png)

```js
const bucket = new WeakMap()
```

调整我们的副作用收集和副作用执行的位置，也就是proxy监听的get和set函数，我们将收集和执行行为单独写成两个函数track和trigger

```js
// 副作用收集函数
function track(target, key) {
  // 没有副作用函数返回
  if (!activeEffect) return
  // 该对象是否有副作用函数
  let depsMap = bucket.get(target)
  if(!depsMap) bucket.set(target, (depsMap = new Map()))
  // 该属性是否有副作用函数
  let deps = depsMap.get(key)
  if(!deps) depsMap.set(key,(deps = new Set()))
  // 将副作用函数和该副作用进行绑定
  deps.add(activeEffect)
}

// 副作用执行函数
function trigger(target, key) {
  const depsMap = bucket.get(target)
  if(!depsMap) return
  const deps = depsMap.get(key)
  if(!deps) return
  deps.forEach(fn=>fn())
}
```

修改代理proxy的set和get函数

```js
const obj = new Proxy({text: 'hello'},{
  get(target, key) {
    // 将副作用函数收集起来
    track(target, key)
    return target[key]
  },
  set(target, key, newValue) {
    const setRes = Reflect.set(...arguments)
    // 将副作用函数从桶中拿出来执行
    trigger(target, key)
    return setRes
  }
})
```

至此我们的整个数据结构如下

![image-20240731155344414](/Users/admin/Desktop/yue-vue/assets/image-20240731155344414.png)

#### 4.4分支切换与cleanup

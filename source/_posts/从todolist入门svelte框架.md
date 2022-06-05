---
title: 从Todolist入门Svelte框架
date: 2021-10-08 15:55:00
updated: 2021-10-08 15:55:00
tags: [front-end,Svelte]
description: 浅谈svelte框架
---

# 从Todolist入门Svelte框架

## Svelte入门

### Svelte-重编译框架-编译器即框架

​  Svelte和React、Vue这些JavaScript框架类似，希望开发者更好的去构建交互式界面，但不同的是Svelte在**构建/编译阶段**将应用程序转换为理想的 JavaScript 应用，而不是在*运行阶段* 解释应用程序的代码。

### Why Svelte?

​  **体积+性能**：Svelte在编译期做静态分析来生成功能，从而减小了打包后得代码体积。Svelte也没有采用Vue、React等流行框架都采用的虚拟DOM而是直接编译生成DOM，可以避免diff操作，理论上性能和手写原生js相同。

​  **上手难度低**：话说，Svelte的官方教程真是相当友好，有中文的官方文档以及入门教程，有聊天室来随时交流问题，甚至还有[Svelte for new developers](https://www.sveltejs.cn/blog/svelte-for-new-developers)这样的详细到让没有使用过node.js的开发者初始化框架的教程。

​  **生态尚未成熟**：生态上不够完善，连一套完整的组件库都没有。如果想要在大型项目中使用Svelte，从考虑长期开发效率和维护角度目前都不是非常好的选择，主流的Vue和React以及angular会是更好的选择，不过目前尚处学生阶段，而Svelte虽是新起之秀不够成熟，也值得开发者学习思想。

​  

​  以上这些都是在大致浏览完Svelte的官方文档以及相关文章后对Svelte的一些看法，然后我会尝试用Svelte写一个TODOList，它会包括基础的增加删除完成以及拓展的修改、回收站、添加删除分组、使用indexdb来缓存历史数据等功能，写TODOList虽然是助手的二面任务，但是我此前也有使用todolist的习惯并且对微软的Microsoft To Do的使用逻辑觉得不太习惯，希望自己写一个demo的时候能让自己满意，简单说我希望做一个比较简约风格的todolist。

## TODOLIST

### 基础的增加、删除、编辑、完成任务功能

需求：todolist的基本功能增删改。

实现：通过对js内数组的增删改并且通过svelte框架的反应性实现实时改变任务列表，再通过svelte的crossfade增加一个简单的动画效果。

### 回收站功能

需求：使用中会有误删的情况，所以增加一个trash bin来防止误删。

实现：通过给对象数组加个成员变量trashed来判断是否处于回收站

### 分组标签

需求：分组标签功能在我此前使用todolist的时候是我认为非常鸡肋的一个功能，虽然绝大多数的todolist都具有分组功能但是还是没有去做这个，在我使用todolist时通常是希望通过todolist做一个短期规划而不是长期规划，来规划我接下来3h或者今天整天或者近几天我希望做的事情，我记录的事情也不会有7件8件那么多，长期规划是确实更需要一个分组标签功能，毕竟总有不同种类的任务需要去区分。

实现：思考过具体如何实现，就是给todos数组加个成员变量tag来区分属于哪个标签组，并且根据对应的tag属性渲染不同的任务区块

### todo状态

需求：点击切换To do/In progress/Paused三种情形

实现：通过svelte框架在html中写if-else判断，点击状态按钮使当前todo对象的状态值改变，然后根据不同的状态值加载不同的html标签，在写的过程中遇到一个神奇的问题

```html
    {#if user.loggedIn}
    <button on:click={toggle}>
     Log out
    </button>
    {/if}
    {#if !user.loggedIn}
    <button on:click={toggle}>
     Log in
    </button>
```

```js
<script>
 let user = { loggedIn: false };

 function toggle() {
  user.loggedIn = !user.loggedIn;
 }
</script>
```

如上写法可以成功通过点击来切换log in/log out，但是这种写法却不行。

```html
    {#if todo.status == "unfinished"}
    <button class="unfinished" on:click={changeStatus(todo)}>
     To do
    </button>
    {:else if todo.status == "inprogress"}
    <button class="inprogress" on:click={changeStatus(todo)}>
     In progress
    </button>
    {:else if todo.status == "paused"}
    <button class="paused" on:click={changeStatus(todo)}>
     Paused
    </button>
    {/if}
```

```js
 //改变状态
 function changeStatus(todo) {
  if (todo.status == 'unfinished'){
   todo.status = 'inprogress';
  }else if (todo.status == 'inprogress'){
   todo.status = 'paused';
  }else if (todo.status == 'paused'){
   todo.status = 'unfinished';
  }
 }
```

通过调试发现能成功通过click事件改变当前todo的status但是这个if判断的逻辑语句却没有办法在变量值改变后去加载改变后的html标签导致无法实现功能，而上面的写法if却可以监测到变量改变，通过调试之后发现可能是这个对象的原因，猜测是我写在todos这个数组里，而在if块的位置todos数组已经加载过了就不会再加载？然后我就尝试了一些别的渲染方法，比如

```html
<button class="status" on:click={changeStatus(todo)}>
 {todo.status}
</button>
```

结果仍然没用，我去查了svelte的文档，看到了反应性能力内更新数组和对象这一块。

**一个简单的经验法则是：被更新的变量的名称必须出现在赋值语句的左侧。例如下面这个：**

```javascript
const foo = obj.foo;
foo.bar = 'baz';
```

**就不会更新对 `obj.foo.bar` 的引用，除非使用 `obj = obj` 方式。**

**我发现因为我的赋值语句是todo.status = 'xxxxx'，因此svelte检测到我更新了点击按钮传进来的todo对象，也就是todos数组的一个元素，但是它检测不到我的todos数组也随之更新了，因此解决方案是在函数内加一句todos = todos，来告诉svelte数组对象更新了，那么它就会随之去更新DOM树了。**

### indexeddb缓存历史数据

需求：因为这是个纯前端实现的方案，而数据如果存在js中那么每次运行项目的数据都没有办法保存，因此想到用indexeddb来做数据缓存。

实现：此前我并没有使用过indexeddb在阅读文档的过程中还是比较生疏，没怎么接触过数据库的内容，大概了解了之后在实际写的过程中还是遇到了相当多的问题，再加上国庆7天因为准备篮球队11月初的省赛早上9-12 下午5-9都在篮球场实在是人比较疲惫吧，做了一半发现不太来的及于是还是先把文章和目前的进度pr上去，等后面有空把这个缓存的功能补上。

### 调整UI

最后对UI进行简单的调整后，demo除了历史数据缓存功能外都完成了。

# 浅谈Svelte框架

​  前端领域是发展迅速，各种轮子层出不穷的行业。最近这些年，随着三大框架`React`、`Vue`、`Angular`版本逐渐稳定，前端技术栈的迭代貌似逐渐缓慢下来，如果将目光放得更加长远来关注谁未来更有可能成为主流技术栈，也许会是新兴的重编译框架Svelte，我希望写一写在我初步了解Svelte后，以Svelte对比主流的前端框架，看一看Svelte产生的背景以及与其他框架对比Svelte的优劣情况。

## Svelte的设计理念

​  `Svelte`作者是 Rich Harris，同时也是 Rollup 的作者。他设计 Svelte 的核心『**通过静态编译减少框架运行时的代码量**』，也就是说Vue 和 react 这类传统的框架，都必须引入运行时 (runtime) 代码，用于虚拟dom、diff 算法。而Svelte直接编译生成DOM,理论上性能和手写原生js相同。Svelte应用所有需要的运行时代码都包含在`bundle.js`里面了，除了引入这个组件本身，你不需要再额外引入一个运行代码。因此Svelte具有体积小、运行速度快等特点。

## Svelte的优势

### 体积小（no runtime）

​  当前的框架无论是 React Angular 还是 Vue，不管你怎么编译，使用的时候必然需要『引入』框架本身，也就是所谓的运行时 (runtime)。，当用户在你的页面进行各种操作改变组件的状态时，框架的运行时会根据新的组件状态计算出哪些DOM节点需要被更新，从而更新视图。这就意味着，框架本身所依赖的代码也会被打包到最终的构建产物中,因此Vue和React等框架打包后的体积相较于Svelte会相对更大。

​  但是用 Svelte 就不一样，一个 Svelte 组件编译了以后，所有需要的运行时代码都包含在里面了，除了引入这个组件本身，你不需要再额外引入一个所谓的框架运行时。

​  下面是`Jacek Schae`的统计，使用市面上主流的框架，来编写同样的Realword 应用的体积：

![svelte bundle size](https://miro.medium.com/max/2000/1*6HK361f-UDqNpWuTA68jHw.png)

### 代码量小

举个例子，`Svelte`中，可以直接使用赋值操作符更新状态，而在`React`中，我们要么使用`useState`钩子，要么使用`setState`设置状态。在此前写Todolist中也我也发现Svelte不需要依赖模板文件，这不仅对代码量来说是减轻对于开发者来说学习成本同样也降低了。

下面是`Jacek Schae`的统计，使用市面上主流的框架，来编写同样的Realword 应用的行数，可以看出Vue和React在代码量上基本齐头并进而Svelte明显要少很多。

![sveltelesscode](https://miro.medium.com/max/2000/1*X6JjPo0y6L02KB7F0RvVlg.png)

### 不错的运行速度

前端框架性能对比，结果中**Svelte 略逊于Vue, 但好于 React。**

![sveltespeed](https://miro.medium.com/max/2000/1*-adYkKBH0YgvRYPp2gbs5Q.png)

## Svelte尚未成熟

​  虽然Svelte具有上述诸多优势，但在开发大型项目时，Svelte没有像AntDesign、ElementUI这样成熟的UI库，原生脚手架没有目录划分，原生不支持预处理器等等，以及于**大型项目必要的单元测试并没有完整的方案**等一系列问题都让目前开发者难以使用Svelte来开发大型项目。

## 如何选型实践

​  Svelte 是否适合在大型项目中应用，还有待观察。虽然核心思想是不需要 “运行时”，但是项目组件越多，运行时的代码量也就越多，且组件间的代码重复率也就越高，除此之外，现阶段的生态确实处于尚未成熟。但是一来作为尚在学校的学生目前接手的项目并不算大型，二来Svelte的优势也确实让人值得相信，或许几年后在诸多开发者的支持下随着Svelte的生态逐步完善，大型项目开发者也会逐渐使用Svelte。

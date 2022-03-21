---
title: 从Todolist入门Svelte框架
date: 2021-10-08 15:55:00
tags: [front-end,Svelte]
---

# 从Todolist入门Svelte框架

## Svelte入门

### Svelte-重编译框架-编译器即框架

​		Svelte和React、Vue这些JavaScript框架类似，希望开发者更好的去构建交互式界面，但不同的是Svelte在**构建/编译阶段**将应用程序转换为理想的 JavaScript 应用，而不是在*运行阶段* 解释应用程序的代码。

### Why Svelte?

​		**体积+性能**：Svelte在编译期做静态分析来生成功能，从而减小了打包后得代码体积。Svelte也没有采用Vue、React等流行框架都采用的虚拟DOM而是直接编译生成DOM，可以避免diff操作，理论上性能和手写原生js相同。

​		**上手难度低**：话说，Svelte的官方教程真是相当友好，有中文的官方文档以及入门教程，有聊天室来随时交流问题，甚至还有[Svelte for new developers](https://www.sveltejs.cn/blog/svelte-for-new-developers)这样的详细到让没有使用过node.js的开发者初始化框架的教程。

​		**生态尚未成熟**：生态上不够完善，连一套完整的组件库都没有。如果想要在大型项目中使用Svelte，从考虑长期开发效率和维护角度目前都不是非常好的选择，主流的Vue和React以及angular会是更好的选择，不过目前尚处学生阶段，而Svelte虽是新起之秀不够成熟，也值得开发者学习思想。

​		

​		以上这些都是在大致浏览完Svelte的官方文档以及相关文章后对Svelte的一些看法，然后我会尝试用Svelte写一个TODOList，它会包括基础的增加删除完成以及拓展的修改、回收站、添加删除分组、使用indexdb来缓存历史数据等功能，写TODOList虽然是助手的二面任务，但是我此前也有使用todolist的习惯并且对微软的Microsoft To Do的使用逻辑觉得不太习惯，希望自己写一个demo的时候能让自己满意，简单说我希望做一个比较简约风格的todolist。

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

### 总结

svelte框架和先前接触的vue+elementUI在使用上还是有一定的相似之处的，关于比较流行的前端库和最近兴起的 Svelte 等重编译时框架，你会如何选择？以及react 等常见框架对于前端有着怎样的意义？不同选型意味着什么？这些问题我会另写一篇博客来谈一下自己的理解吧，此篇用于记录以制作todolist的demo来入门svelte框架的过程。
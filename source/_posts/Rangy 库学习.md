---
title: Rangy 库学习
date: 2017-08-17 00:55:22
tags: JavaScript
---


最近想给自己的阅读器加上注释的功能，一番折腾之后发现 [Rangy](https://github.com/timdown/rangy)  这个好用的库。使用这个库可以便捷地实现获取和保存文本选取，以及对文本选取添加样式等功能。下面让我们来初步了解一下它的用法，更详细的内容可以查阅它的文档 [Here](https://github.com/timdown/rangy/wiki) 。

<!-- more -->

&nbsp;

## Selection 和 Range 对象

首先我们需要了解 `Selection` 和 `Range` 这两个和文本选取有关的对象。

Selection 对象表示用户选择的文本范围或插入符号的当前位置。它代表页面中的文本选区，可能横跨多个元素。文本选区由用户拖拽鼠标经过文字而产生。要获取用于检查或修改的 Selection 对象，请调用  [`window.getSelection()`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/getSelection)。

```js
var sel = window.getSelection(); // 获取原生 Selection 对象
ver text = sel.toString(); // 获取选中文本
var range  = sel.getRangeAt(0); // 获取对应的 Range 对象
```

&nbsp;

一个 `Selection` 对象可能包含多个 `Range` 对象，通常只包含一个。因为用户通常下只能选择一个范围，但是有的时候用户也有可能选择多个范围（例如当用户按下 Control 按键并框选多个区域时，Chrome 中禁止了这个操作，原译者注）。

&nbsp;

> **注意**：Rangy 库根据 DOM 规范自己实现了一个 Selection 对象和 Range 对象，下面方法的 API 中的 Selection 对象和 Range 对象都是指自己实现的对象。



```js
var sel = rangy.getSelection(); // 获取 **自定义** 的 Selection 对象

// --- 获取 iframe 的 Selection 对象
var iframe = document.getElementById("foo");
var sel = rangy.getSelection(iframe);

// --- 选中某个节点
var el = document.getElementById("foo");
var range = rangy.createRange();
range.selectNodeContents(el);
var sel = rangy.getSelection();
sel.setSingleRange(range);
```

&nbsp;

>  **注意**：在下面的 API 里，一个或多个函数或方法参数周围的方括号表示参数是可选的。

&nbsp;

## Serializer 模块

Serializer Module 可以对 `Selection` 对象和 `Range` 对象进行序列化和反序列化，有了它之后我们就可以实现注释的保存。

这个模块可以将鼠标选中的文本区域序列化为一个字符串进行存储，并在需要的时候反序列化以恢复成文本被鼠标选中的模样。

值得我们注意的是 `Selection` 对象的保存于 DOM 树密切相关，因此序列化和反序列化是比较“脆弱的”，对于动态页面可能不能做到正常的序列化和反序列化。另外，在一个浏览器上进行的序列化结果，可能不能在另一个浏览器被正常反序列化。

下面是两个常用的方法：

* `rangy.serializeSelection([Selection sel[, Boolean omitChecksum, [Element rootNode]])` 
  * 不传入参数时，默认序列化当前页面的选中文本区域；传入参数 `sel` 时，会对传入的这个 `Selection` 对象进行序列化。
  * 参数 `omitChecksum` 表示是否省略校验和，这个在文档很大的时候建议把它设置为 `true` 以加快速度。—— **注**：“校验和”是为了弥补上面所说的“脆弱性”而进行的校验计算。
  * 参数是 `rootNode` ，如字面意思“根节点”。Serializer Module 会处理“根节点”下的被选中的文本区域，将它们序列化。默认值是当前文档的根节点。
  * 方法最后会返回一个字符串，代表选中部分
* `rangy.deserializeSelection(String serializedSelection[, Element rootNode[, Window win]])` 
  * 接收上一个方法返回的字符串作为参数，反序列化。
  * 其余参数和上面方法参数类似
  * 返回一个 `Selection` 对象
* `rangy.serializeRange(Range range[, Boolean omitChecksum, [Element rootNode]])` / `rangy.deserializeRange(String serializedRange[, Element rootNode[, Document doc]])` 与上面两个方法类似，是对 `Range` 对象序列化和反序列化的版本。



对于上述方法的使用我们可以看看下面这个简单的例子：

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<title>Document</title>
</head>
<body>
<div>
	<p>
		Lorem ipsum dolor sit amet, consectetur adipisicing elit. Voluptate quidem, tempore. Cupiditate veritatis, doloribus provident quae impedit odit alias minus accusamus distinctio corrupti sapiente dolorum eius, voluptates dolores ipsa veniam.
	</p>
</div>
<input type="button" value="serialize" onclick="ser()">
<input type="button" value="deserialize" onclick="dser()">

<script type="text/javascript" src="../../lib/rangy-core.js"></script>
<script type="text/javascript" src="../../lib/rangy-serializer.js"></script>
<script type="text/javascript">
	var text = '';
	var sel = null;
    
	function ser() {
        sel = rangy.getSelection();
		text = rangy.serializeSelection(sel);
		console.log(text);
	}
	function dser() {
		sel = rangy.deserializeSelection(text);
		console.log(sel);
	}
	window.onload = function () {
		rangy.init(); // 必要的初始化
	};
</script>
</body>
</html>
```

```js
// 针对 iframe 的情况
var iframe = document.getElementById("foo");
var iframeWin = iframe.contentWindow || iframe.contentDocument.defaultView;
var selection = rangy.getSelection(iframe); // 获取 iframe 的 Selection 对象
var serializeSelection = rangy.serializeSelection(selection);
rangy.deserializeSelection(serializeSelection, null, iframeWin);
```

&nbsp;

## Class Applier 模块

上面我们已经实现了获取文本选取和保存恢复文本选取的功能了，最后一步就是对选取应用样式了（比如说高亮选取部分）。

我们可以使用 Class Applier 模块，这个模块允许我们使用特定的标签包裹已经选取的文本，并且对此标签应用在 CSS  中已定义样式。

&nbsp;

使用的方法如下：

* `rangy.createClassApplier(String theClass[, Object options[, Array tagNames]])` 
  * 第一个参数 `theClass` 必须是在 CSS 中声明过的一个类，不可少。
  * 第二个参数 `options` 是个配置对象，允许你对 class applier 的行为进行操作，主要包含下面的属性：
    * `elementTagName` ：string，默认为 `"span"`，表示注释标签 。class applier 会使用 `<span>` 标签包裹用户选中的区域，并对 `<span>` 标签应用 `theClass` 这个类，于是结果会变成这样：`<span class="类名">用户选中的部分</span>` 。
    * `elementProperties` ：object。这个属性值是一个对象，对象的每个属性和值，都会变成注释标签的属性和值。如：`elementProperties: { href: '#'}` ，这个配置会让生成的注释标签变成这样 `<span href="#"></span>` 。**注意**：这个属性很方便，甚至可以用 `elementProperties: { onclick: function () {} }` 来挂载回调函数。
    * `onElementCreate` ： 如名字所示，当注释标签被创建时调用的回调函数。回调函数接收两个参数：第一个是对创建标签的引用，第二个是当前的 class applier 对象。
  * 函数返回一个 `ClassApplier ` 对象，使用这个对象可以调用下面的方法来为选中区域添加或删除 `theClass` 定义的样式。
* `applyToSelection([Window win])` 
  * 为选中内容添加预设的样式，不提供参数时默认监听当前的窗口；也可以提供一个 `Window ` 对象（比如操作 `iframe` 时可以传入 `iframe.contentWindow` ，下同）
* `undoToSelection([Window win])` 
  * 消除 class applier 为选中内容添加的样式。
* `isAppliedToSelection([Window win])` 
  * 判断选中部分 **都** 被 class applier 添加了预设的样式
* `toggleSelection([Window win])` 
  * 有则添加，无则删除预设的样式
* `applyToRange(Range range)` / `undoToRange(Range range)` / `isAppliedToRange(Range range)` / `isAppliedToRange(Range range)` / `toggleRange(Range range)` 
  * 作用和上面对应方法类似

&nbsp;

对于上述方法的使用我们可以看看下面这个简单的例子：

```html
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>Document</title>
	<style>
      	// 预设的样式
		.red {
			background-color: red;
		}
	</style>
</head>
<body>
<div>
	<p>
		Lorem ipsum dolor sit amet, consectetur adipisicing elit. Voluptate quidem, tempore. Cupiditate veritatis, doloribus provident quae impedit odit alias minus accusamus distinctio corrupti sapiente dolorum eius, voluptates dolores ipsa veniam.
	</p>
</div>
<input type="button" value="class applier" onclick="classApply()">
<input type="button" value="remove class" onclick="classRemove()">

<script type="text/javascript" src="../../lib/rangy-core.js"></script>
<script type="text/javascript" src="../../lib/rangy-classapplier.js"></script>
<script type="text/javascript">
	var applier;

	function classApply() {
		applier.applyToSelection(); // 为选中部分应用样式
	}

	function classRemove() {
		applier.undoToSelection(); // 取消选中部分的样式
	}

	window.onload = function () {
		rangy.init(); // 必要的初始化
		applier = rangy.createClassApplier("red"); // 创建 ClassApplier
	};
</script>
</body>
</html>
```

&nbsp;

## 参考

* [MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Selection)
* [Rangy wiki](https://github.com/timdown/rangy/wiki) 
* [stackoverflow](https://stackoverflow.com/questions/11586115/range-deserializeselection-checksum-error)
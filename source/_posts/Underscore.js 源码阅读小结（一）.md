---
title: Underscore.js 源码阅读小结（一）
tags: Underscore.js
date: 2017-04-29 16:02:00
---

> 源码版本 1.4.4

## 创建 Underscore 对象的引用

```js
var _ = function(obj) {
    if (obj instanceof _) return obj;
    if (!(this instanceof _)) return new _(obj);
    this._wrapped = obj;
};
```

上面的代码创建了 undersore 的引用，然而我对执行顺序却理得不是很清楚，尤其是上面的第三行，于是我尝试在 chrome 中 debug 了一下。不过在理解之前我们先复习一下使用 `new` 调用构造函数的具体流程，以及 `instanceof` 操作符的用法。

### new 调用构造函数的实际操作

我们都知道，用 `new` 调用构造函数时，操作符会创建一个空对象，然后让被调用函数的 `this` 指向这个空对象。这样，在构造函数内部使用的 `this.property` 就可以为这个空对象添加属性了。同时，`new` 会为空对象添加一个属性 `__proto__`，让此属性指向构造函数的原型对象。调用构造函数之后，`new` 会将这个空对象返回，这个空对象就是构造函数的实例对象了。

为了理解，我们来看下面的这段代码，它调用构造函数生成了 Color 的实例对象：

```js
function Color () {
    this.name = 'red'; // 此处给空对象添加属性
}
var foo = new Color(); // foo 为函数的实例对象
```

在上面的代码中，我们用 `new` 调用构造函数时，它实际执行的操作如下：

```js
var obj  = {}; // 创建一个空对象 obj
obj.__proto__ = Color.prototype; // obj 的 __proto__ 属性指向构造函数的原型对象
Color.call(obj); // 将构造函数的 this 与 obj 绑定，在调用构造函数
foo = obj; // 将 obj 返回
```

从上面可以清晰地看到， `new` 创建了一个空对象，并为其添加了 `__proto__` 属性，然后将函数的 `this` 绑定到此对象上执行函数，最后返回了此空对象。

### instanceof 操作符的作用

`instanceof` 运算符用来判断一个构造函数的 `prototype` 属性所指向的对象是否存在另外一个要检测对象的原型链上。

这句话有点抽象，但是我们知道通常我们可以用 `instanceof` 操作符来判断一个对象的类型。在上面出现的代码中，如果我们执行这一行代码 `foo instanceof Color` ，会发现结果为 `true`，因为 foo 的类型即为 Color。

现在我们再考虑这句话，发现实例对象 foo 上拥有一个指向 Color 的原型对象的 `__proto__` 属性，因此 `instanceof` 操作符的结果为 `true`。这就是 `instanceof` 操作符能够判断对象类型的原因。

回顾了 `new` 与 `instanceof` 的用法，接下来我们来看在创建一个具体的 Underscore 对象时，构造函数的执行过程。

### 构造函数的执行过程

由于构造函数本身也是一个函数，因此我们有如下两种调用方法来创建对象：

```js
var _ = function(obj) {
    if (obj instanceof _) return obj;
    if (!(this instanceof _)) return new _(obj);
    this._wrapped = obj;
};

var bar = new _({}); // 方法一
var foo = _({}); // 方法二
```

当我们使用方法一创建 `_` 的实例对象时，`new` 操作符会像上面提到的那样创建一个空对象，空对象的 `__proto__` 属性指向 `_.prototype`，然后执行构造函数。

在构造函数的第二行，由于 `obj` 实际为 `{}`，不是 `_` 的实例，结果为 `false`，接着执行第三行。在第三行中，由于 `new` 创建的空对象已经包含了指向 `_.prototype` 的 `__proto__` 属性，且 此函数的 this 就指向这个空对象，于是 `this instanceof _` 结果为 `true`，再取反后为 `false`，于是跳过执行第四行。最终在第四行把 `obj` 交给 `_` 的 `_wrapped` 属性。

从上面可以看出，使用第一种方法创建 `_` 的实例时，第三行貌似可以删除，因为总会跳过。但是，当我们用第二种方法生成 `_` 的实例是，第三行就比不可少了。

使用方法二直接在全局中调用构造函数时，构造函数的 `this` 指向外部的全局对象 `window`。此时当代码执行到第三行后，由于 `window` 不包含指向 `_.prototype` 的属性，`!(this instanceof _)` 会返回 `true`，于是执行 `return new _(obj)`。这样就会像第一种方法一样，会把 `obj` 交给 `_wrapped`，生成一个 `_` 的实例对象。这个实例对象最终由 `return` 返回给 `foo`。

简言之，第三行是为了让我们能够用第二种方式创建 `_` 对象实例而存在的。

## Array.prototype.forEach() 方法

语法：

```js
array.forEach(callback(currentValue, index, array){
    //do something
}, this)
```

`callback` 为数组中每个元素执行的函数，该函数接收三个参数：`currentValue`(当前值)数组中正在处理的当前元素。`index`(索引)数组中正在处理的当前元素的索引。`array`为 forEach() 方法正在操作的数组。`thisArg`可选可选参数。当执行回调 函数时用作 this 的值(参考对象)。

（一直以来都还不知道`thisArg`这个参数呢 = =）

## _.each(obj, iterator, [context])

`_.each()` 方法用于遍历一个数组或者对象，类似于 ECMAScript 5 为 Array 提供的原生的 `forEach()` 方法，只不过同样适用于对象而已。如果存在原生的 forEach 方法，Underscore 会优先调用原生的方法。

```js
// 一个特殊的对象，这个对象用于控制在循环中跳出
var breaker = {};

// 参数：
// obj 为要遍历的对象
// iterator 处理函数
// context 为处理函数的 this 绑定的对象
var each = _.each = _.forEach = function (obj, iterator, context) {
    if (obj == null) return;
    if (nativeForEach && obj.forEach === nativeForEach) {
        obj.forEach(iterator, context); // 若存在原生的方法，使用原生的方法遍历
    } else if (obj.length === +obj.length) {
        for (var i = 0, l = obj.length; i < l; i++) { // 遍历，对每一项调用 iterator 来处理
            if (iterator.call(context, obj[i], i, obj) === breaker) return;
        }
    } else {
        for (var key in obj) { // 遍历对象的属性，对每个属性调用 iterator 处理
            if (_.has(obj, key)) { // _.has 是后面定义的方法，判断 key 是不是对象本身拥有的（即 key 不是原型上的）
                if (iterator.call(context, obj[key], key, obj) === breaker) return;
            }
        }
    }
};
```

在代码的第五行有一句 `obj.length === +obj.length`，它的作用是判断对象 `obj` 是否拥有 `length` 属性（判断 `obj` 是不是数组或类数组对象）。在这里，会先执行 `+obj.length`，`+` 的作用是把 `obj.length` 转化成数字类型；然后 `===` 将转化的结果与 `obj.length` 相比较，看两者的类型和值是否相同。

一般来说, `Object` 类型是没有 `length` 属性的，因此 `obj.length` 是 `undefined`，`+obj.length` 是 `NaN`，比较结果为 `false`。而对于 `Array`、`String` 类型以及其他类数组对象来说，它们都是拥有 `length` 属性的，而且可以用方括号来访问对应位置的值，因此可以在后面用 `for` 循环来遍历。

在第十一行有一个 `breaker`，它是在前面定义的一个变量，其值为 `{}`。关于 `breaker` 的作用，正如注释里所提到的那样，是用来跳出 `each` 的循环的。在 Underscore.js 的其他地方，会有另外的一些函数调用 `each` 函数。当这些函数想要从 `each` 循环中跳出时，它们就会返回一个 `breaker`。这样第 19 行的结果会为 `true`，`each` 函数就停止执行了。

如果还没有理解的话可以看看下面的代码，它是 Underscore.js 里的 `some()` 函数的简化版。`some()` 函数的作用是判断数组中是否有元素满足迭代器的检测，只要任意一个元素满足了检测就返回 true，并中断对数组的遍历。

```js
_.some = function (obj, iterator) {
    
    var result = false;
    if (obj == null) return result;
    
    // 这里用 each 函数遍历元素
    each(obj, function (value, index, list) {
        // 若任一元素满足iterator的检测，则返回一个breaker
        if ((result = iterator(value))) 
            return breaker; // 返回了breaker，each函数会停止执行，循环结束
    });
    return result;
};
```

### 类数组对象

在 js 的 DOM 操作中，经常遇到一个 `NodeList` 属性，它就是一个类数组对象。对于类数组对象，我们可以用 `length` 来访问它的元素个数，还可以用方括号语法来访问对应下标的值。

其实，在上面第 9 行代码处就是借用了类数组对象。就算 `obj` 本身不是数组，像 `String` 类型也可以用方括号和下标访问对应地方的值，因此可以用 for 循环来遍历。

了解更多关于类数组对象的内容可以看看[这里](http://www.open-open.com/lib/view/open1462621480193.html)。

## _.bind(function, object, *arguments)

函数 `bind` 作用与 ES5 提供的原生的 `bind` 函数作用相同：将函数 `function` 绑定到参数 `object` 上，即函数无论在何处被调用，内部的 `this` 始终指向绑定的 `object`。`bind` 的方法的第三个可选参数 `arguments` 是提前传给 `function` 的任意数量的参数，这样，函数在调用时只接收剩余未绑定的参数。`bind` 方法返回原函数的拷贝函数，这个函数的 `this` 及部分参数已被指定。

下面 Underscore.js 中对这个方法的实现：

```js
// 用于原型设置的可重用构造函数
var ctor = function(){};

_.bind = function (func, context) {
    var args, bound;
    if (func.bind === nativeBind && nativeBind) return nativeBind.apply(func, slice.call(arguments, 1)); // 原生bind存在则使用原生的方法
    if (!_.isFunction(func)) throw new TypeError; // func不是函数则抛出错误
    args = slice.call(arguments, 2); // 以数组形式保存传入bind的可选参数*arguments
    
    // 返回一个闭包，用bound保存返回的函数
    return bound = function () {
        
        if (!(this instanceof bound)) return func.apply(context, args.concat(slice.call(arguments))); // this不是bound的实例时，调用func
        
        ctor.prototype = func.prototype; // ctor暂时保存func的原型对象
        var self = new ctor; // new一个ctor的实例（没有属性，但有指向func原型的指针）
        ctor.prototype = null; // 删除ctor上保存的func的原型对象
        var result = func.apply(self, args.concat(slice.call(arguments))); // 调用func
        if (Object(result) === result) return result;
        return self;
    };
};
```

初次看这段源码时对第13行内的 `if` 判断及相应的代码很困惑，在网上查找一番后终于在 segmentfault 上找到了[答案](https://segmentfault.com/q/1010000002551122)。

原来，当一个函数以普通方式调用时，即 `f()`，`this` 会指向 `window` 对象。此时的 `!(this instanceof bound)` 自然为 true，进入 `if` 内部执行。但是，如果函数被用 `new` 调用时，即 `new f()`，此时 `!(this instanceof bound)` 语句中的 `this` 有一个指向 `bound` 原型对象的指针 `__proto__`（详见前面提到的new与instanceof操作符的实际操作），因此这句的结果为 false，执行 `if` 后面的内容。

`if` 后面的代码就是针对使用 `new` 调用函数的情况。我们知道，使用 `new` 调用函数时，函数的 `this` 指向一个空对象。如果上面的代码没有针对 `new` 调用函数的特殊情况做处理，那么函数的 `this` 会指向绑定的 `context`，就出错了。
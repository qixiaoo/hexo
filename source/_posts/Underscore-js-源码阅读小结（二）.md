---
title: Underscore.js 源码阅读小结（二）
date: 2017-05-05 17:33:47
tags: Underscore.js
---

> 源码版本 1.4.4

## Array.prototype.reduce()

ECMAScript 5 提供了两个归并数组的方法：`reduce()` 和 `reduceRight()`。这两个方法都会迭代数组的所有项，然后构建一个最终返回的值。其中，reduce() 方法从数组的第一项开始，逐个遍历到最后。而 reduceRight() 则从数组的最后一项开始，向前遍历到第一项。 

`arr.reduce(callback,[initialValue])` 接收两个参数，第一个参数为一个回调函数，数组中的每个元素都会被传入此函数中迭代；第二个参数为一个可选的初始值，它的值会被作为回调函数的第一个参数传入。

这里重点在于理解 reduce() 的第一个参数 `callback`，它是一个回调函数，这个回调函数接收四个参数：

* accumulator 上一次调用回调返回的值，或者是提供的初始值（initialValue）
* currentValue 数组中正在处理的元素
* currentIndex 数组中正在处理的元素索引，如果提供了 initialValue ，从0开始；否则从1开始
* array 调用 reduce 的数组

<!-- more -->

让我们通过下面数组求和的示例理解 reduce() 函数：

```js
var sum = [0, 1, 2, 3].reduce(function(acc, val) {
  return acc + val;
}, 0); // 实际操作为 (0+0)+1)+2)+3

console.log(sum); // 6
```

执行上面的代码后，会在控制台输出6，即数组求和的结果。这里的执行流程是这样的：数组调用 reduce() 后，首先会把初始值0作为回调函数的第一个参数传入，然后取数组的第一个元素（即0）作为回调函数的第二个参数传入，再调用回调函数，会执行 `0+0` 得到结果0，回调函数最后返回这个结果。这样一次迭代就完成了，然后会开始第二次迭代。第二次迭代时，reduce() 会把前一次回调函数返回的结果（即0）作为此次迭代中回调函数的第一个参数，然后取数组的第二个元素（即1）作为回调函数的第二个参数，再调用回调函数，会执行 `0+1` 得到结果1，回调函数再返回这个结果。reduce() 函数会反复进行这个过程，直到数组中最后一个元素也被迭代处理，reduce() 最后返回累加的结果。

因为 reduce() 函数的第二个参数是可选的，所以当我们未传入第二个参数时，reduce() 会把数组的第一个元素作为初始值传给回调函数，然后从数组的第二个元素开始迭代，直到数组的最后一个元素也被处理。示例代码如下：

```js
var sum = [0, 1, 2, 3].reduce(function(acc, val) {
  return acc + val;
}); // 未传入初始值，实际操作为 (0+1)+2)+3

console.log(sum); // 6
```

上面的代码就相当于把数组的第一项作为初始值，然后从数组的第二项开始迭代，最后返回迭代的结果。

如果数组为空并且没有提供 initialValue， 会抛出 `TypeError`。如果数组仅有一个元素（无论位置如何）并且没有提供 initialValue， 或者有提供 initialValue 但是数组为空，那么此唯一值将被返回并且 callback 不会被执行。

**注意**，无论是 reduce() 还是 reduceRight() 函数都是属于数组的。因此，如果我们想要用 reduce() 迭代处理对象的所有属性的话，我们需要使用 Underscore.js 的 `_.reduce()` 函数。

## _.reduce(list, iteratee, [memo], [context])

Underscore.js 的 `reduce()` 函数的作用与前文中提到的原生的 reduce() 的作用相同。两者的不同之处在于 Underscore.js 提供的 reduce() 函数功能更强大，它还可以迭代处理对象的所有属性。

list 是需要迭代的对象，可以是一个数组或对象；iteratee 是迭代函数；memo 是 reduce() 函数的初始值；context 是 iteratee 的 this 绑定的值。reduce() 的每一步都需要由 iteratee 返回。这个迭代传递4个参数：memo, value 和迭代的 index（或者 key）和最后一个引用的整个 list。

下面 Underscore.js 中对这个方法的实现：

```js
  // 如果数组为空并且没有提供 initialValue， 会抛出 TypeError
  var reduceError = 'Reduce of empty array with no initial value';

  // 迭代处理对象的成员并返还一个单一的值。 若 ECMAScript 5 原生支持reduce，
  // 将调用原生的reduce函数。 此函数也有别名inject和foldl。
  _.reduce = _.foldl = _.inject = function (obj, iterator, memo, context) {
      var initial = arguments.length > 2; // 大于2说明传入了初始值memo
      if (obj == null) obj = [];
      if (nativeReduce && obj.reduce === nativeReduce) { // 判断原生方法是否存在
          if (context) iterator = _.bind(iterator, context); // iterator的this与context绑定
          return initial ? obj.reduce(iterator, memo) : obj.reduce(iterator); // //调用原生的reduce
      }
      each(obj, function (value, index, list) { // 用已定义的each函数迭代处理obj的每个元素
          if (!initial) { // 若未提供初始值memo，则第一次迭代时会进入内部
              memo = value; // 若未传入初始值，把obj的第一项作为初始值
              initial = true; // 修改initial的值，确保each的下一次迭代不会再进入此if语句内
          } else {
              // 用memo保存每次迭代的结果，并作为下一次的初始值传入iterator
              memo = iterator.call(context, memo, value, index, list);
          }
      });
      if (!initial) throw new TypeError(reduceError); // 上面提到的异常情况
      return memo;
  };
```

这个 reduce() 函数的实现还是挺简单的。若存在原生的方法，则调用原生的 redece()。否则就用 `_.each()` 函数迭代 `obj` 的每一项元素，并用 `memo` 保存每次迭代的结果，再进行下一次迭代处理。

## \_.some(list, [predicate], [context])与\_.find(list, predicate, [context])

`_.some()` 函数的作用是判断 `list` 内是否存在元素满足 `predicate` 的检测。其中，`list` 可以是一个数组或者对象，`predicate` 是一个回调函数。回调函数要求返回 true 或 false 来表示当前元素是否满足检测。`_.find()` 的功能与 `_.some()` 相似，不过它是返回满足 `predicate` 检测的第一个元素。

**注意**，这两个函数都是在找到第一个满足条件的元素后就立即停止对 `list` 的遍历。

下面让我们来看一看源码：

```js
// Underscore.js 中默认的迭代器
_.identity = function (value) {
    return value;
};

var any = _.some = _.any = function (obj, iterator, context) {
    iterator || (iterator = _.identity); // 若未提供检测函数，则使用默认的identity
    var result = false;
    if (obj == null) return result;
    if (nativeSome && obj.some === nativeSome) return obj.some(iterator, context); // 优先使用原生的方法
    each(obj, function (value, index, list) {
        if (result || (result = iterator.call(context, value, index, list))) 
            return breaker; // 找到满足条件的元素后就中断遍历
    });
    return !!result; // 确保返回的是布尔型元素
};

_.find = _.detect = function (obj, iterator, context) {
    var result;
    any(obj, function (value, index, list) {
        if (iterator.call(context, value, index, list)) {
            result = value; // 保存找到的第一个满足条件的元素
            return true;
        }
    });
    return result;
};
```

上面的源码里有两个需要我们注意的地方。

第一个是在第13行中匿名函数的内部，当我们找到了一个满足条件的元素后，匿名函数返回了一个 `breaker`。这个我们在上一篇文章讲 `_.each()` 函数的时候提到了，返回 `breaker` 是为了终止 `_.each()` 函数的执行，即停止遍历元素。

另外一个是在第15行中使用了一个 `!!result`。我们知道对一个布尔型变量进行两次取反后，它的值是不变的。这里之所以要对 result 两次取反，是为了确保返回的是布尔型元素。从第 7 行可以看出，如果我们没有提供检测函数，那么 `_.some()` 就会使用默认的 `_.identity`。然而 `_.identity` 是不会对元素进行处理的，它会直接返回当前遍历到的元素。因此我们使用两次取反来对元素进行类型转化，让元素变成布尔型变量。
---
title: Godot 中的内存泄漏
date: 2021-01-09 16:25:38
tags: Godot
---

最近在使用 Godot 的过程中踩了一个删除节点的坑，因此查阅了一些资料，下面是一点记录。

**注意**，本人并不是 Godot 专家，下面仅仅是开发过程中发现的一些问题以及一点总结，如果存在错误的话欢迎指正。

<!-- more -->

&nbsp;

### Orphan Nodes

Orphan Nodes 即孤儿节点，指游离在内存中没有父节点的节点。在一个正常的游戏中，存在着一定数量的孤儿节点是正常的。比如，如果我们实现了一个对象池，那么池中未被使用的节点就会是孤儿节点。

对象池中的节点虽然没有父节点，但实际上还是能够被代码访问到的，因此不能说是产生了内存泄漏。不过如果一个孤儿节点不能够通过代码访问到了，那么此时就产生了内存泄漏。

如果发现随着游戏进行或者场景切换，孤儿节点有增无减，那么大概率有可能是发生了内存泄漏。此时应当排查是不是哪里的代码有问题。一般来说，孤儿节点产生的原因都是我们在编写代码时，忘记删除不再被使用的节点实例导致的。

Godot 编辑器提供了查看孤儿节点的调试工具。

在代码中调用 `print_stray_nodes()` 可以在控制台中打印出当前游戏中存在的孤儿节点。

```gd
func _ready():
	var orphan = Node.new()
	orphan.name = "orphan node"
	print_stray_nodes()

# output:
# 1293 - Stray Node: orphan node (Type: Node)
```

此外，在游戏运行状态下，打开 “调试器” - “监视” 面板，然后勾选 “Object” - “Orphan Nodes”，也可以实时查看当前游戏中存在的孤儿节点的数量。

{% asset_img "孤儿节点.png" "孤儿节点" %}

&nbsp;

### remove\_child 与孤儿节点

调用 `remove_child` 方法来动态地移除子节点可能是产生孤儿节点的原因之一。API Doc 对此方法的描述如下：

> Removes a child node. The node is NOT deleted and must be deleted manually.

`remove_child` 方法只是把子节点从其父节点的子节点列表中移除，不会自动将被移除的子节点从内存中删除。因此，如果我们在代码中只是手动通过 `remove_child` 移除了子节点，那么被移除的子节点将会成为一个孤儿节点。

当确认一个节点不再被使用时，我们需要手动调用 `Object` 对象上 `free` 方法，或者 `Node` 对象上 `queue_free` 方法来确保该节点被删除。

```gd
func _ready():
	var orphan = Node.new()
	orphan.name = "orphan node"
	orphan.free()
	print_stray_nodes()
```

&nbsp;

### Object 类与垃圾回收

对于上面 Godot 提供的 `remove_child` 方法的行为，习惯了使用 JavaScript 来操作 DOM 节点的开发者可能会感到诧异。

JavaScript 引擎提供了一套垃圾回收机制，引擎会在对象不能够被访问到时自动将其删除。垃圾回收机制让我们写代码更加舒适，不用耗费脑力在内存管理上面。不过有得有失，实现垃圾回收机制通常都会牺牲一定的性能。

Godot 没有垃圾回收机制，因此你需要手动管理内存。

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">A lot of new Godot users ask about object pooling...<br>You don&#39;t need to do that in Godot because allocating/freeing scenes/classes is fast and there is no garbage collector in GDScript/Godot.<br>I wonder how we could best un-educate new Godot users about this practice..</p>&mdash; Juan Linietsky (@reduzio) <a href="https://twitter.com/reduzio/status/1073284242086551552?ref_src=twsrc%5Etfw">December 13, 2018</a></blockquote>

&nbsp;

Godot 中所有非 built-in 类型都继承自 `Object` 类。查看 API Doc 可以得知 `Object` 的实例是没有被垃圾回收机制管理的。

> Objects do not manage memory. If a class inherits from Object, you will have to delete instances of it manually. To do so, call the free() method from your script or delete the instance from C++.

`Object` 以及其子类需要自己来管理内存，在实例不再被使用时需要手动将其 `free` 掉。

我们日常在开发中经常会使用到两种 `Object` 的子类：`Node` 和 `Reference`。

正如前面提到的，`Node` 及其子类的实例在不被使用时需要手动删除。因此，日常我们通过 `.new ()/.instance()` 动态创建的各种 `Node`（如 Node2D，Control 等）都需要手动管理其内存。

而 `Reference` 则相对比较特殊，它实现了[引用计数](https://zh.wikipedia.org/wiki/%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0)。这意味着 `Reference` 以及其子类的实例不需要我们手动管理其内存。`Resource` 类型就是 `Reference` 的一个子类，因此我们动态创建地各种 `Texture` 之类的资源在不使用时是不需要手动删除的。

在大多数情况下，如果我们想要自定义一些 class，可以选择继承 `Reference`，这样会自动使用引用计数来管理内存。
---
title: Godot 场景树与节点
date: 2019-06-02 19:12:23
tags: Godot
---

[Godot](<https://godotengine.org/>) 是一款开源的游戏引擎，同时支持 2D 和 3D 游戏的开发。相对于 Unity 3D 来说，Godot 的学习曲线更加平缓，对于初学者来说是一个不错的选择。本文是本人最近阅读 Godot 官方文档和查阅相关资料后的一点个人理解，如果有错误的地方还请各位朋友指出。

<!-- more -->

&nbsp;

Godot 引擎的执行流程如下。在一开始时先根据所属的平台来创建一个 `OS` 类的实例。然后执行必要的初始化工作，加载所有驱动程序、服务器、脚本语言、场景系统等。初始化完成后，需要向 `OS` 提供一个 `MainLoop` 来运行。我们编写的游戏的所有节点的逻辑都是在引擎的主循环中执行的。

游戏所有运行中的节点组成了一棵树，我们称其为场景树（`SceneTree`）。场景树 `SceneTree` 是 `MainLoop` 的子类，里面包含的逻辑会在程序启动后被游戏引擎在主循环中执行。我们可以通过调用当前场景树中任意节点的 `get_tree()` 方法来获取当前场景树的实例对象（`SceneTree` 的实例）。

场景树的实例有两个比较重要的属性，`root` 和 `current_scene` 属性。这两个属性分别代表当前**场景树的根节点**和当前运行着的**游戏场景的根节点**。

**值得注意是**，场景树并不是当前运行着的游戏场景的所有节点组成的树。场景树不仅包含了当前运行中的游戏场景节点，也包含了游戏项目中被设置为[自动加载](<https://docs.godotengine.org/zh_CN/latest/getting_started/step_by_step/singletons_autoload.html>)的单例节点。因此如果我们在日志窗口中打印出场景树的结构，可以看到场景树根节点 root 的直接子节点中，包含着自动加载的节点和当前游戏场景根节点。

```gd
 ┖╴root # 根节点
    ┠╴Game # 自动加载的单例节点
    ┠╴Util # 自动加载的单例节点
    ┠╴DialogManager # 自动加载的单例节点
    ┖╴Node # 当前游戏场景根节点
       ┠╴ColorRect1
       ┠╴ColorRect2
```

由于自动加载的节点在游戏运行启动时是优先加载的，因此可以看到自动加载的节点排在当前游戏场景根节点的前面。

&nbsp;

## root 节点

场景树的根节点 root 是一个 `Viewport` 对象。由于 `Viewport` 类型是派生子 `Node` 类型的，因此在调试时可以通过 `Node` 类型的 `print_tree_pretty()` 方法来打印出当前场景树的结构。上面的场景树结构就是调用这个方法打印出来的。

```gd
get_tree().root.print_tree_pretty()
```

当然，Godot 编辑器也提供了更为直观的查看场景树中节点的方法。在游戏运行时，点击“场景”标签页下“远程”按钮，就可以看到当前场景树中的各个节点。

{% asset_img "查看当前场景树状态.png" "查看当前场景树状态" %}

上图中 root 节点下的前三个子节点都是我项目中“自动加载”（AutoLoad）的单例节点，第四个子节点是正在运行着的游戏场景的根节点。一般来说，游戏场景的加载晚于自动加载的节点的加载，因此当前游戏场景的根节点总是 root 节点的最后一个子节点。所以在官方示例代码中，有时候我们可以看到使用下面的代码来获取当前游戏场景的根节点。

```gd
var root = get_tree().root
var current_scene = root.get_child(root.get_child_count() -1)

# 也可以这么获取当前游戏场景的根节点
get_tree().current_scene
```

&nbsp;

Viewport 在官方文档中被翻译为视口，即屏幕上的一块可视区域。场景树的根节点 root 是一个 `Viewport` 对象，游戏场景中的所有内容都是绘制在这上面的。可以利用 `Viewport` 提供的方法来实现游戏场景截图，如下：

```gd
var img = get_tree().root.get_texture().get_data()
img.flip_y()
img.save_png(path) # path 为截图保存的路径
```

比较有趣的是，调用 `get_texture()` 方法返回的是一个 `ViewportTexture` 对象。这个纹理对象上的内容与当前视口上呈现图像是同步的，这就意味着如果你把这个对象赋值给 `Sprite` 展示，那么就可以实现画中画的效果。

```gd
var texture = get_tree().root.get_texture()
var sprite = Sprite.new()
sprite.texture = texture
```

{% asset_img "画中画效果.gif" "画中画效果" %}

&nbsp;

## SceneTree 常用 API

下面是一些 `SceneTree ` 可能会比较频繁地用到的 API。

`root` 和 `current_scene` 属性。作用见上文。

`pause` 属性。`pause` 属性是布尔类型变量，使用它可以比较轻松地实现[暂停游戏](<https://docs.godotengine.org/zh_CN/latest/tutorials/misc/pausing_games.html>)的功能。当将 `SceneTree` 的 `pause` 属性设置为 `true` 时，场景将会暂停物理模拟、事件处理以及 `_process/_physics_process` 回调。结合暂停白名单节点，可以实现自由暂停和恢复游戏。

```
get_tree().paused = true
```

&nbsp;

`call_group()` 与 `call_group_flags()` 方法。这两个方法可以签名如下，使用它们可以调用所有属于同一组的节点的特定方法。

```gd
Variant call_group( String group, String method, ... )
Variant call_group_flags( int flags, String group, String method, ... )
```

从方法参数可以看出两个方法的差异在于 `flag` 参数。`flag` 参数说明了在调用特定方法时的行为，可以取如下的值。

- `GROUP_CALL_DEFAULT = 0` 默认情况，空闲帧时调用方法
- `GROUP_CALL_REVERSE = 1` 按节点出现的顺序的逆序依次调用方法
- `GROUP_CALL_REALTIME = 2` 立即调用方法
- `GROUP_CALL_UNIQUE = 4` 只调用依次方法，即使 call_group 执行了多次

&nbsp;

与上面这一对方法类似的还有 `notify_group()` / `notify_group_flags()`，`set_group()` / `set_group_flags()`。它们的作用分别是向组内节点发出通知和设置组内节点的属性值。其它与组相关的方法还有 `get_nodes_in_group()` / `has_group()`，分别用于获取组内节点和判断某个组是否存在。

&nbsp;

与场景相关的方法 `change_scene()`，`change_scene_to()`，`reload_current_scene()`。方法签名如下，前两者用于切换场景，第三个用于重置当前场景。

```gd
Error change_scene( String path )
Error change_scene_to( PackedScene packed_scene )
Errorreload_current_scene()
```

&nbsp;

`quit()` 和 `set_quit_on_go_back()` 方法。调用前者可以直接退出游戏，设置后者为 `true` 会使 Android 用户点击 back 键时直接退出游戏。

```gd
void quit()
void set_quit_on_go_back( bool enabled )
```

&nbsp;

信号：

- `screen_resized` 屏幕尺寸变化时触发
- `tree_changed` 场景树的结构变动时触发
- `node_added/node_removed` 节点变动时触发
- `idle_frame/physics_frame` `_process/_physics_process` 调用前触发
- `files_dropped` 文件被拖放到游戏窗口上触发

&nbsp;

一个延迟到下一个空闲帧执行的小技巧（来自交流群群友）：

```gd
yield(get_tree(), 'idle_frame')
# your code
```

&nbsp;

## Node 常用 API

在介绍节点常用的 API 之前有一点需要强调，即**节点只有进入了场景树后才会被激活**，未激活的节点将不具备节点的诸多功能。如下，我们用 `Timer` 节点实现一个 `set_timeout` 方法。只有将动态创建的 `Timer` 节点加入到了场景树中之后，`Timer` 节点才会在超时时发出 timeout 信号。

```gd
func set_timeout(obj: Object, method: String, time: float) -> void:
	var timer := Timer.new()
	timer.one_shot = true
	timer.autostart = false
	timer.wait_time = time
	
	add_child(timer) # 需要将节点加入到场景树中，激活节点
	timer.start()
	
	yield(timer, 'timeout')
	obj.call(method)
	timer.queue_free()
```

&nbsp;

当节点进入场景树时，它们将变为活动状态。它们可以访问他们需要处理的所有内容，获取输入，显示 2D 和 3D 视觉效果，接收和发送通知，播放声音等。当他们从场景树中删除时，他们将失去这些能力。

Godot 中的大多数节点操作（例如绘制 2D，处理或获取通知）都按树顺序完成。这意味着树顺序中具有较低等级的父母和兄弟姐妹将在当前节点之前得到通知。存在继承关系时，父节点相关操作会先执行。

&nbsp;

`name` 属性代表节点的名字。当我们调用 `print_tree_pretty()` 方法时，可以看到场景树中的各个节点都被打印出来了，且同一父元素的子节点的名字是各不相同的。一般情况下我们不需要设置节点的名字。不过如果节点是动态创建并加入到树中的时候，那么这个节点是匿名的，我们可以手动为其设置一个名字（不是必须的）。

```gd
add_child(Timer.new())
get_tree().root.print_tree_pretty()

# output
... ...
 ┠╴CanvasLayer
       ┃  ┖╴Sprite
       ┠╴Dialog
       ┃  ┠╴Label
       ┃  ┖╴AnimationPlayer
       ┖╴@@2 # 没有名字的 Timer 节点
```

&nbsp;

`pause_mode` 属性规定了当节点所属的 `SceneTree` 的 `pause` 属性设置为 `true` 之后节点自身的行为。此值默认为 `PAUSE_MODE_INHERIT`，表示节点随场景树一起暂停。当此值为 `PAUSE_MODE_STOP` 时，表示无论什么情况，节点都暂停。当此值为 `PAUSE_MODE_PROCESS` 时，表示无论什么情况，节点都不暂停。

`_enter_tree()` / `_exit_tree()` / `_ready()` / `_process()` / `_input()`，生命周期内的各种方法，在合适的时候被调用。

&nbsp;

其它常用的节点处理方法如下。

```gd
# 子节点处理方法
void add_child( Node node, bool legible_unique_name=false ) # 插入子节点
Node get_child( int idx ) const # 获取指定索引子节点
int get_child_count() const # 获取子节点数目
Array get_children() const # 获取子节点列表

Node get_parent() const # 获取父节点
Node Pathget_path() const # 获取节点路径
Node Pathget_path_to( Node node ) const # 获取节点相对路径
Node get_node( NodePath path ) const # 根据路径查询节点

SceneTree get_tree() const # 获取场景树
Viewport get_viewport() const # 获取视口

void move_child( Node child_node, int to_position ) # 删除子节点
void queue_free() # 删除自己

void print_tree_pretty() # 打印自己与子孙节点
```


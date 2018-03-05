---
title: 学习在网页中使用 SVG
date: 2018-03-05 19:58:29
tags: 前端
---

好久没有写博客了，上一篇文章居然还是暑假的时候……

为了避免博客被荒废掉，并且在这学期勉励自己，特地在此发出本学期的第一篇文章。今天我们来说一下在网页中使用 SVG。

SVG 的全称是“可缩放矢量图形”，而所谓的矢量图形简言之就是可以任意放大和缩小的图形。而我们平时使用手机拍照得到的和电脑上常常操作的以 `.jpg`、`.png`，`.gif` 为后缀的图片则统统是“位图”，位图是由一个一个的像素所组成的图形。如果在平时你有放大过位图的经历的话，你会发现当图片放大到一定程度后就会变得十分模糊，而 SVG 图片是可以无限放大而不失真的。SVG 图片以 `.svg` 为后缀。

## 了解 SVG

为了在网页中使用 SVG 图片，我们首先需要了解基本的 SVG 图片是什么样的。我们先来看看怎样创建一张 SVG 图片。

首先你需要在桌面新建一个文本文档，然后在文档内容中贴入下面的代码，然后保存，最后将文本文档的后缀名需改为 `.svg` 即可。这样我们就成功绘制了一张矢量图啦！

代码内容如下：

```svg
<svg xmlns="http://www.w3.org/2000/svg"
    xmlns:xlink="http://www.w3.org/1999/xlink">

    <circle cx="40" cy="40" r="24" style="stroke:#006600; fill:#00cc00"/>

</svg>
```

创建后如果我们要查看这张矢量图长什么样的话，我们可以直接使用浏览器打开这个文件。在浏览器中我们可以看到图像是这样的：

{% asset_img "circle.png" "一个使用 SVG 绘制的圆" %}

很简单的吧。

不过我们之前贴的那段代码是什么意思呢？我们现在仔细观察我们贴入文本文档中的代码，有没有感觉很熟悉？对了，我们可以看到上面的那段 SVG 代码和我们平时写的 HTML 的标签非常像。

没错。我们在浏览器中看到的丰富多彩的矢量图其实都是由像上面那样的一个一个的标签所构成的，比如每个 SVG 文件都有的根标签 `<svg>` 和上面代码中代表圆形的标签 `<circle>`。

事实上，SVG 规范定义了相当多的标签供我们使用，比如基本的形状标签 `<rect>`、`<ellipse>`、`<line>`，以及更高级复杂的路径标签 `<path>` 等等。因此我们可以使用这些标签来绘制各种各样的图片。

{% asset_img "Low-Poly.jpg" "SVG 绘制的 Low Poly 风格图片" %}

至于学习基本的 SVG 标签的使用，我们可以参考下面两篇教程。

第一篇是 Google 排行靠前的一本 GitBook [SVG入门教程](https://www.gitbook.com/book/brucewar/svg-tutorial/details)，第二篇是 MDN 的 [SVG教程](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Tutorial)。基本看完任意一篇后就会使用基本的 SVG 元素了。

此外在学习时，为了避免每次查看效果都要新建一个文本文档，我们可以使用 [CodePen](https://codepen.io/) 或者 [JS Bin](http://jsbin.com) 来实时查看 SVG 图片的效果。

## 基本的使用

在对 SVG 有了大概的了解之后，我们就可以尝试在网页中使用 SVG 图片了。我们至少有两种方式使用 SVG 图片。

其一是使用 `<image>` 标签，并将其 `src` 属性置为 SVG 图片的路径；其二则是直接将代表图片的 SVG 标签放在网页中。如下所示：

```html
<!-- 方法一 -->
<img src="/svg/circle-element-1.svg">

<!-- 方法二 -->
<div>
    <svg>
        <circle cx="40" cy="40" r="24" style="stroke:#006600; fill:#00cc00"/>
    </svg>
</div>
```

平常我们使用时，可以到网上去下载我们需要的资源。另外，我们也可以使用软件 AI 或者 Inkscape 自己动手绘制。

## SVG 动画

个人学习 SVG 以来，感觉最有趣的部分就是 SVG 动画了。当然，这也是最难的一部分。

SVG 规范中提供了一系列标签来为 SVG 图形添加动画，包含 `<set>`、`<animate>`、`<animateColor>`、`<animateTransform>` 和 `<animateMotion>`。讲道理，要彻底弄清楚这些标签的各个属性还是很难的（至少我还没记住），学习的时候可以看看 [张鑫旭大神的博客](http://www.zhangxinxu.com/wordpress/)。

虽然 SVG 规范提供了这么多的动画标签，但是我们平时一些常用的动画其实都可以使用 CSS3 来完成。比如说线条动画和颜色动画。

<p data-height="265" data-theme-id="0" data-slug-hash="PQNzKw" data-default-tab="html,result" data-user="bubble-Q" data-embed-version="2" data-pen-title="smile" class="codepen">See the Pen <a href="https://codepen.io/bubble-Q/pen/PQNzKw/">smile</a> by bubble-Q (<a href="https://codepen.io/bubble-Q">@bubble-Q</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

另外，其他较为复杂的动画我们可以借助相关的 js 库来完成。比如 AlloyTeam 出品的 [pasition](https://github.com/AlloyTeam/pasition) 和 Adobe 的 [Snap.svg](http://snapsvg.io/)。

其实如果平时有兴趣的话，我们甚至可以学学 AI 或者 Inkscape 什么的。这样的话，我们有创意时就可以自己实现了。下面是我用 AI 画的一张小小的作品。

<p data-height="265" data-theme-id="0" data-slug-hash="BYeOrM" data-default-tab="html,result" data-user="bubble-Q" data-embed-version="2" data-pen-title="SVG Windmills" class="codepen">See the Pen <a href="https://codepen.io/bubble-Q/pen/BYeOrM/">SVG Windmills</a> by bubble-Q (<a href="https://codepen.io/bubble-Q">@bubble-Q</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

## 其他相关工具

差不多上面这些就是全部的内容了。另外，下面还有两个比较有用的相关小工具，空闲的时候玩玩也挺不错的。

首先是位图转矢量图工具 [Vector Magic](https://vectormagic.com/)，它可以把你的位图转换为矢量图。

然后就是 Low Poly 风格图片制作工具 [Image Triangulator](http://www.conceptfarm.ca/2013/portfolio/image-triangulator/)，它可以通过人工添加锚点的方式来轻松制作一张 Low Poly 风格的图片。

以上。
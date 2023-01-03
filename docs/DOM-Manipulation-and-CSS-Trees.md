#  DOM 操作和 CSS 树

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/DOM-Manipulation-and-CSS-Trees>

我们都知道，像 Windows 的文件资源管理器中看到的树，只不过是一个嵌套的列表。但是有可能在 HTML/CSS 中将一棵树编码成嵌套列表吗？

让我们从列表开始。这是一个标准文件树的片段，组织在“文件夹”中。

```
<ul>
 <li>
  Graphics
  <ul>
   <li><a href='gpu.h'>gpu.h: Function prototypes</a></li>
   <li><a href='gpu.c'>gpu.c: Graphic output implementation</a></li>
   <li>
    Debug Output
    <ul>
     <li><a href='dbgout.h'>dbgout.h: Output prototypes</a></li>
     <li><a href='dbgout.c'>dbgout.c: Output fixed-width font drawing</a></li>
     <li><a href='font5x7.h'>font5x7.h: Fixed-width font definitions</a></li>
    </ul>
   </li>
  </ul>
 </li>
</ul>
```

这是我们从代码中得到的。

*   制图法
    *   [gpu.h:函数原型](gpu.h)
    *   [gpu.c:图形输出实现](gpu.c)
    *   调试输出
        *   [dbgout.h:输出原型](dbgout.h)
        *   [dbgout.c:输出定宽字体图](dbgout.c)
        *   [font5x7.h:固定宽度字体定义](font5x7.h)

拥有树意味着每个分支节点都可以展开或折叠，以显示或隐藏树中的元素。展示和隐藏并不困难；如果我们定义两个类，CSS 中的`display`属性允许我们非常快速地完成这项工作:

```
ul.hide { display: none; }
ul.show { display: block; }
```

当然，事情没那么简单。你不能非常容易地改变一个`UL`的状态(点击它不行)；但是你可以改变一个`LI`的状态。所以我们把这两个类移到封闭的李:

```
li.hide ul { display: none; }
li.show ul { display: block; }
```

所以，我们有隐藏树的 CSS。但是我们如何转换状态呢？如何才能随意显示和隐藏节点？这就是 DOM 出现的原因。如果我们将树项目的描述(例如“调试输出”)放在一个活动元素中(`DIV`或`A`)，我们可以将 DOM 事件附加到它上面。

我已经决定不使用`A`，因为一个锚需要一个`href`，并且使用一个`#`的链接会使你的浏览器的历史功能变得混乱。所以，我们用一个`DIV`。

当我们点击`DIV`时，我们希望发生什么？基本上，只需翻转父`LI`的状态，这样下面的`UL`就可见了。

```
<ul>
 <li class='hide'>
  <div onclick='toggle(this.parentNode)'>Graphics</div>
  <ul>
   <li><a href='gpu.h'>gpu.h: Function prototypes</a></li>
   <li><a href='gpu.c'>gpu.c: Graphic output implementation</a></li>
   <li class='hide'>
    <div onclick='toggle(this.parentNode)'>Debug Output</div>
    <ul>
     <li><a href='dbgout.h'>dbgout.h: Output prototypes</a></li>
     <li><a href='dbgout.c'>dbgout.c: Output fixed-width font drawing</a></li>
     <li><a href='font5x7.h'>font5x7.h: Fixed-width font definitions</a></li>
    </ul>
   </li>
  </ul>
 </li>
</ul>
```

所以当你点击“图形”`DIV`，`toggle()`运行，将顶部的`LI`从`hide`翻转到`show`。当然，如果你再次点击它，它会跳回`hide`。我们需要一些 JavaScript 来完成这项工作；幸运的是，JS 提供了三元运算符，我们可以根据条件选择两个选项。

```
function toggle(x)
{
  x.className = (x.className=='show') ? 'hide' : 'show';
}
```

这意味着:“如果类名是`show`，就设置为`hide`，否则【即。如果不是`show`，设置为`show`。由于类名只有两种可能，您可以看到这两者之间的切换。

在我们开始实际的工作示例之前，您应该记住，您必须为我们进入的每一级菜单定义 CSS，因为如果有一个`LI`挡道(总是存在)，属性不会在`UL`之间继承。

所以现在，我们可以把所有这些放在一起，并提出一个简单的树，它是可折叠的/可扩展的，只需要一点点 DOM 改动。

*   图形
    *   [gpu.h:函数原型](gpu.h)
    *   [gpu.c:图形输出实现](gpu.c)
    *   调试输出
        *   [dbgout.h:输出原型](dbgout.h)
        *   [dbgout.c:输出定宽字体图](dbgout.c)
        *   [font5x7.h:固定宽度字体定义](font5x7.h)

在这里，我刚刚给文本`DIV`添加了一些样式，它可以像`UL`一样随着父`LI`状态而改变。同样，属性的继承会在不同级别之间丢失，所以只需为每个级别添加一行。

现在我们只有一个问题。如您所见，页面加载时，图形项目处于折叠状态。没有 JS 运行怎么办？点击`DIV`，无任何反应；下面的单子你拿不到！显然是个问题。缓解这种情况的方法是默认展开所有内容，而不是折叠；如果需要，可以使用`onload`树折叠，这样如果运行 JS，树就会折叠，如果不运行 JS，树就会保持展开状态。

```
function treeCollapse(){
  var list = document.getElementById('yourtree').getElementsByTagName('li');
  for(var i=0;i<list.length;i++) list[i].className = 'hide';
}
```

这将得到树中所有`LI`的列表，并将它们设置为类`hide`。

所以，这就是如何把一个嵌套列表做成一个可扩展的树。

文章日期:2006 年 9 月 22 日
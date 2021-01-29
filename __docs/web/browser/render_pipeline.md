# 渲染流水线

!> 本文讲述内容基于 Chromium 的机制

你或许已经知道了怎么用 HTML 写页面结构，用 CSS 写样式，然后他们就会被浏览器变成最终变成显示屏上展示的图像，变成一个又一个的像素点。<u>那么HTML和CSS是如何转换成显示器能显示的内容的呢？</u>

![](https://cdn.jsdelivr.net/gh/arronKler/oss@master/uPic/2020_12/4grqJq_10_11-52-05.jpg ':size=100%')

这中间的转换过程，其实就是一个 **“渲染流水线”** 的执行过程。所谓渲染流水线 是指浏览器中的渲染进程将我们的html、css 代码转换为屏幕上显示的像素点的过程。之所以叫做流水线，是因为整个渲染的过程和工厂中的流水线类似，按顺序一步一步执行，最终产出结果。

首先，我们先看一下完整的渲染流水线：
1. 构建DOM（Build DOM）
2. 计算样式（Style Calculation）
3. 布局（Layout）
4. 分层（Layer）
5. 绘制（Paint）
6. 分块（Tile）
7. 栅格化（Raster）
8. 合成（Composite）


当我们访问一个网页的时候，浏览器的网络进程会下载网页的数据，当发现是HTML文档（Response Header 中的 Content-Type 为 text/html），网络进程会通知浏览器进程创建对应的渲染进程，渲染进程创建好之后会和网络进程之间建立一个数据通道。当网络进程开始接收 HTML 文档的数据的时候会 **流式** 的将接收到的数据塞入数据通道中给到渲染进程进行处理。渲染进程在这个时候就开始了渲染流水线的执行。

> 流式： 表示并不需要等到所有数据都下载完才传递，而是下载多少就传递多少

!> 对浏览器的进程感兴趣的，可以参见我的另一篇文章 [浏览器进程架构的演化](/web/browser/browser_process.md)

HTML、CSS 经过上面整个渲染流水线的过程，执行下来，就变成了屏幕上的图像了，接着我们详细探讨各个流程

# 构建DOM

> 什么是DOM？ DOM就是文档对象模型，它是浏览器内部表示页面的数据结构，也是用户操作页面的接口。整个页面的渲染流程是从拥有一颗DOM树开始的。

!> 【参考】HTML语法解析标准规范文档：[https://html.spec.whatwg.org/multipage/parsing.html#overview-of-the-parsing-model](https://html.spec.whatwg.org/multipage/parsing.html#overview-of-the-parsing-model)

我们先看一下 HTML 标准中定义的构建DOM的流程：

![HTML转换为DOM的规范流程](https://cdn.jsdelivr.net/gh/arronKler/oss@master/uPic/2020_12/4Yw1Nq_10_11-53-09.jpg)

上面这张图描述了DOM树的构建流程，首先，从网络获取到字节流的数据，然后将字节流的数据解码成输入的字符流，并对输入的字符流做预处理。字符流经过 tokenizer 做词法分析，将文本解析为一个个的词 (token)，然后根据词来构建DOM树。🌲 在构建DOM树的过程中，遇到 script 脚本，还会先暂停构建，先去执行脚本，然后再继续 DOM 树的构建流程，直到所有文本流数据都被解析为最终的 DOM 树。

其中有两个点我们需要特别讲解一下：
1. **tokenizer 做词法分析**。将文本字符串转换为一个个的词 (token)
2. **创建DOM**。基于已经分析好的token，借助辅助栈构建 DOM 树


## tokenizer
字符流自然是不被计算机所理解的，要能被理解，第一步就是要根据 **词法规则** 转换为一个个的 **词（token）**，这个过程也可以叫做词法分析。这个过程是由 **分词器（tokenizer）** 来做的。

> 词法规则：词法规则由HTML标准进行定义

比如如下 HTML 

```html
<html>
<head>
  <title>Document</title>
</head>
<body>
  content
</body>
</html>
```

就会被解析为
![](https://cdn.jsdelivr.net/gh/arronKler/oss@master/uPic/2020_12/aowqDg_10_11-52-28.jpg ':size=100%')

!> tokenizer 一般是由状态机来实现的

## 创建DOM

通过tokenizer分词器进行转换后，单纯的文本信息变成了序列化的一个个token，这时候我们就可以按顺序一个个的分析 token 创建节点，并添加到 DOM 树中。直到所有的 token 都被解析完成，DOM就构建好了。

构建DOM的构成中，

创建DOM的规则：
1. token为 StartTag： 创建新节点，加入DOM树中。该 token 入栈
2. token为 EndTag：从辅助栈中取栈顶元素，对比如果是匹配的就出栈该token，DOM树的指针返回上一层
3. token为文本：创建文本节点，加入DOM树中

注意要点：
1. JS阻塞DOM解析。遇到 Script 标签，会去执行 JS，停止DOM解析
2. 遇到`<link ...>`、`<image ...>` 等资源给到网络进程去下载

!> DOM的创建有个细节就是借助了辅助栈来进行构建

### JS 阻塞
在构建DOM的过程中，如果遇到了 script 标签，会阻塞DOM的解析，分两种情况
1. script标签中包含js代码。直接给到 js 引擎执行代码
2. 如果是引用了外部js文件。先给到网络进行下载，然后再执行代码

不管是哪种情况，DOM 解析都会暂停，等待 js 执行完成后才恢复 DOM 的解析。哪怕是对于第二种情况，也是要等它下载完成，执行完成，才会恢复 DOM 解析。

Why stop?

因为脚本的执行可能会改变 DOM 树的结构，比如下面的代码

```html
<script>
 document.write('<p>');
</script>
```


### CSS阻塞
在进行JS阻塞的时候，如果存在未处理完成的CSS信息。比如外部链接的CSS文件正在下载中、或者CSS内容已经有了，但是还没有解析成完整的CSSOM。那么此时的CSS会阻塞 JS 的执行。

Why?

js的执行可能会操作样式。考虑如下情况，当我们在JS中执行改变某个元素样式的代码

```javascript
element.style.color = 'red'
```

此时，如果css没有解析完成，我们不等待css的解析，那么即使这里的js执行了，也会被后续css的解析结果给覆盖掉。

而且我们需要注意的是，因为我们没法在执行 js 代码的时候就确定他会改变样式（还没执行，当然不知道js干了些啥）。所以，<u style="color: red">**只要是遇到script标签要做js执行的时候，若存在没有执行完成的css任务，css任务都会阻塞当前js的执行，直到它执行完成，js才继续执行，最后再继续解析DOM**</u>

## 优化：预解析线程
渲染引擎拿到数据的时候会启用一个与解析线程，用来分析是否存在引用外部链接的 js、css文件，然后提前下载这些文件。


# 计算样式


# 布局


# 分层

# 绘制

# 分块

# 栅格化

# 合成
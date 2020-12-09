
## 什么是sourcemap

我们在写javascript代码的时候，往往在上线之前会将其打包或者压缩，目的有这些

1. 打包多个文件为一个文件，减少HTTP请求
2. 压缩文件体积

等等。

但是打包压缩之后，在浏览器中运行时，如果一旦出错，那它将会提示的错误信息就不是我们源文件中的对应错误位置，而是打包压缩后的js文件执行时错误的位置，就像这样

![errorinfo](https://b-gold-cdn.xitu.io/v3/static/img/frontend.1dae74a.png)

打包压缩后的文件一般是常人看不懂的，那这种错误提示信息也就没什么作用了，因为我们只能知道是程序报错了，但是在源文件中到底是哪儿错了，就全然不知了。特别是线上环境中如果一旦报错，查起来就很费劲了。

那又想使用打包压缩这样的功能，又不想遇到错误提示的时候难以排查错误信息，怎么办呢？这个时候sourcemap的出现就是一个必然了，正如名字所说，sourcemap中存储了打包后的文件与源文件中位置和变量信息的映射。这样的话，在压缩打包后的文件运行时出现错误提示的时候，只要通过sourcemap文件，就能找到源文件中对应的实际错误位置，然后给出源文件中的错误位置信息，这样就非常利于开发人员既能很好的时候打包压缩的功能，又大可放心出错信息难以解读的问题了。

## sourcemap是怎么和打包后的文件关联起来的

用实例来说，我们现在新建一个目录，然后初始化一个项目，加上webpack，

```bash
$mkdir test
$cd test
$npm init -y
$npm install webpack webpack-cli --save-dev
```

初始化目录结构

```bash
$mkdir src
$mkdir dist
$touch webpack.config.js
$touch src/index.js
```

在package.json中加入

```json
{
  "scripts": {
    "start": "webpack"
  }
}
```

在webpack.config.js 中写入

```javascript
// webpack.config.js
module.exports = {
  devtool: 'cheap-source-map',
  mode: 'production'
}
```

然后在 src/index.js 中写入一些代码

```javascript
// src/index
function One() {
  console.log('nice job');
}
```
准备就绪，在命令行中运行

```bash
$npm start
```

打包完成后，命令行中会出现类似这样的信息

![输出图片](http://localhost/img.png)

这里可以它看到在dist目录下一共生成了两个文件，main.js 和 main.js.map ，有map后缀的这个文件就是生成的sourcemap文件了。我们先打开main.js并把光标移到文件末尾，会发现这样一行语句

```javascript
//# sourceMappingURL=main.js.map
```

这也就代表着，当前的这个main.js用了这样的一个注释表示它对应的sourcemap文件是main.js.map，在浏览器中执行这个main.js的时候，浏览器就知道了： 哦，你的sourcemap文件是这个，我可以通过这个文件在给你提示错误的时候先匹配一下，然后给出你的真实错误信息了。

所以，有了打包的时候生成的sourcemap文件，有了打包文件对它所对应的sourcemap文件的一个注释，这一切就成为了可能。

## sourcemap原理是什么

sourcemap其实本质就是一个JSON文件，包含了 version、sources、names、file、mappings 等信息。类似于这样：

```json
{
  "version":3,
  "file":"main.js",
  "names":[
     "installedModules",
     "__webpack_require__",
     "moduleId",
     "exports",
     "module",
     "i",
     "l",
     "modules",
     "call",
     "m"
  ],
  "sources":[
      "webpack:///main.js"
  ],
  "mappings":"AAAAA,AbcAA;BBBBB",
  "sourceRoot":""
}
```

其中file表示的是当前打包后的文件；sources是一个数组，包含了一个或多个打包前的源文件；还有names是变量标识符的一个列表；
其中最重要的就是mappings了，它整个就是一个字符串，我们把它叫做位置编码字符串，也就是说它记录了位置，而且这个位置是经过编码的。我们来看看它是怎么编码的。

我们在上面的示例sourcemap文件的mappings字段写的是 `AAAAA,AbcAA;BBBBB` ，实际你操作的时候这个字符串会特别长，为了便于理解我就写这么一个作为示例。

可能你会想一个文件中那么多位置，怎么通过一个字符串就记录下来了。我们会发现mappings字段中的内容其实是有一定的规律性的，它会先出现连续几个字符，然后是逗号(`,`)，重复这个过程，然后有时会遇到分号(`;`)。它们表示了什么？

- 分号(`;`) 分隔了打包后文件的行，它表示 `行对应` 。
- 逗号(`,`) 分隔了位置记录字符串，表示 `位置对应` , 也即分隔多个具体的位置匹配记录
- `AAAAA` 这样的就是针对某个具体位置的 `位置匹配编码` 了。

类似 `AAAAA` 这种的，每个这样的字符串都表示了一个完整的信息 —— **“打包后代码中的哪一列对应了sources中哪个源文件文件中的哪一行的哪一列的哪个标识符”**。听起来有点绕是不是，其实关键信息是如下几部分：

- 打包后文件中的哪列。（分号(`;`)已经告诉我们是哪行了）
- sources中的哪个源文件
- 源文件中的哪行
- 源文件中的哪列
- 对应的变量名是啥(因为有的时候错误提示并非有具体是变量的错误，所以这个信息有时候也可省略)

也就是说，每个逗号分隔出来的字符串其实是由5个部分按顺序放在一起组成的，也就是上面我列出来的这5个部分，比如第一个分号前的 `AAAAA`，代表的是 —— “在打包文件中的第0行的第0列（第一个A），对应了sources数组中的第0个文件(第二个A)中的第0行(第三个A)第0列(第4个A)的 names数组中的第0个标识符(第五个A)”

这里我们做了一个假设就是A对应的数字是0。实际编码的过程中，我们这5部分的原始信息实际上都是一个位置数值，但是位置数值不好统一化的进行处理，有的数值很大有的很小。为了提高存储效率，我们对每部分信息我们都用了一个叫做VLQ的编码方式，将原本的数值通过编码变成了一个或多个字符。如果你还想深入，我们继续说VLQ。

# 变量与函数声明


## var


## let和const


## 函数
这其中最有趣的就是函数了，

下面代码执行结果是啥？
```javascript
function foo() {
  console.log(bar)
}

foo()
```

报错 ，`ReferenceError: bar is not defined` 对吧，那么如果这样呢?

```javascript
function foo() {
  console.log(bar)

  if (false) {
    function bar() {}
  }
}

foo()
```

结果是 `undefined` 。为啥？

这里其实是块级作用域的问题，bar处在 if 的代码块中，如果就在if块中定义和执行，其结果跟在一个函数里面执行是一样的。但如果在块外去引用了，此时就有意思了。

```javascript
function foo(…) {
    …
    {
         …
         function bar(…) { … }
         …
    }
    …
}

// 等价于
function foo(…) {
    var bar0 = undefined; // function-scoped
    …
    { // 注意这里是最顶部，其上没有任何其他语句
         let bar1 = function bar(…) { … }; // block-scoped
         …
         bar0 = bar1;
         …
    }
    …
}
```

# 解读执行上下文、环境记录、作用域链

!> 本文基于 EcmaScript 2020 版本的标准规范


## 执行上下文

### 执行上下文是什么？
什么是执行上下文呢？请看下面的官方文档给出的解释：

> An execution context is a specification device that is used to track the runtime evaluation of code by an ECMAScript implementation.

从定义上来说，<u>执行上下文是用来在代码运行期间，跟踪代码的运行时状态的机制</u>。其实咱们可以这么理解，执行上下文，指的就是执行期间的上下文信息。什么是上下文信息？比如一个函数在运行的时候，传入的参数，声明的内部变量及函数，this值，作用域等等，都是代码执行期间js引擎需要给当前代码提供的上下文信息。没有这些信息，代码就没法正常执行。

因此，js引擎在执行一段script脚本代码的时候，需要先创建一个全局的执行上下文。它在执行一个函数的时候，也会先给这个函数创建一个函数的执行上下文。

执行上下文分类：
1. 全局上下文
2. 函数上下文
3. Eval上下文

### 它是如何管理的？

> The execution context stack is used to track execution contexts. The running execution context is always the top element of this stack. A new execution context is created whenever control is transferred from the executable code associated with the currently running execution context to executable code that is not associated with that execution context. 

执行上下文在内存中是 “栈” 式管理的，被称为 <u>**执行上下文栈**</u>，也被叫做**调用栈**。当script中的代码进行执行的时候，会先创建一个全局上下文，推入上下文栈中。然后如果执行了函数，就会创建这个函数所需要的执行上下文，并推入栈中，这时候这个函数的执行上下文就是栈顶元素。标准中也规范了，处于栈顶的就是 **运行中的执行上下文**。那么，无论我们在执行哪个函数，当前执行上文栈的栈顶元素刚好就是我正在执行的函数所关联的上下文，当函数执行完了需要退出的时候，弹出栈顶的元素，整个程序的运行状态就切换回了“外层代码块”的执行过程当中了。

比如下面这段代码：

```javascript
// EC: 全局上下文

function foo() {
  // EC: foo 函数执行上下文

  function bar() {
    // EC： bar 函数执行上下文
    console.log('bar')
  }

  bar()
  console.log('foo')
}

foo()

```

![execution context](https://cdn.jsdelivr.net/gh/arronKler/oss@master/uPic/2020_12/execution_context_stack_17_14-41-34.png ':size=100%')

其执行上下文栈的变化如同上图所示，还没有执行的时候上下文栈的是空的，开始执行整个代码的时候会先创建好全局执行上下文，比如在浏览器中的window对象就是在其全局执行上下文中提供的。执行foo的时候，foo的上下文入栈，再接着bar的上下文入栈。bar执行完成后，出栈。此时恢复foo函数的执行，执行完后出栈退回到全局执行上下文下，没有代码了，全局执行上下文也出栈，结束程序运行。

!> 调用栈是有大小的，如果无限调用下去，就会出现 stack overflow 栈溢出 


### 执行上下文里面有什么？
前面说到执行上下文会保存一些当前代码执行所需的必要信息。你可能已经猜到了，我们声明的变量、函数等也是在执行上下文中提供的。


```
EnviromentRecord = {
  [[ThisValue]]
  [[ThisStatus]]
  [[OuterEnv]] // must have
}

ExecutionContext = {
  [[code evaluation state]]
  [[Function]]
  [[Realm]]
  [[ScriptOrModule]]

  // ECMAScript code only 
  [[LexicalEnvironment]] = EnvironmentRecord
  [[VariableEnvironment]] = EnvironmentRecord
}
```

## 环境记录


## 作用域链


## 总结

- 执行上下文：给代码运行期间提供必要信息的机制
- 执行上下文栈：用来跟踪执行上下文的变化。栈顶元素就是当前正在运行的代码的执行上下文
- 执行上下文的分类：
  - 函数上下文
  - 全局上下文
  - Eval上下文
- 
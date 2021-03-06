---
title: JavaScript异常处理最佳实践
date: 2017-3-11 15:53:08
tags:
- javascript
---

自己在写JavaScript的时候经常会遇到如何处理错误的问题

- 要不要检查一下参数是不是正确的？
- 如果参数不正确是抛出异常，还是传给`callback`，还是静默让它执行？
- 我该如何编写检查的代码技能保证代码的精练又能达到检查的目标？

<!-- more -->

并且，如果每个函数都写一下参数参数那程序本身就会变得很臃肿，而且违背了 JavaScript 这门语言本身的定位。
除了上面的问题，我们还会有这些问题：

- 如果出现异常了该抛出异常，还是传给`callback`，还是触发`EventEmitter`等等
- 如何提供足够的信息让调用者能够知道错误的细节？
- 如何处理未预料的错误？

## 1 错误分类

我们把错误分成两大类

### 1.1 操作失败

操作失败并不是由于代码的原因，而是这个操作本身就可能会出现错误。比如说连接数据库，可能由于网络环境出现暂时的连接错误。读取一个文件，但这个文件不存在。这种错误是无法避免的，但应该要有意识地处理这种错误。

### 1.2 错误代码
比如说需要传递一个数字时传递了一个字符串进去、变量名拼写错误等等。这种情况属于代码错误，我们应当在开发阶段就完全排除所有代码错误。

## 2 错误处理

不同的错误方式，相应的处理方式也不同。

### 2.1 代码错误

对于错误代码造成的错误，最好的处理方式就是什么都不做，抛出错误，让程序崩溃。这样子在开发的时候就能够发现错误。同时你也不应该指望通过编写其他代码去处理那些本身就有错误的代码。

### 2.2 操作失败

对于正常操作发生的错误，我们可以做以下事情

 - 直接处理，比如尝试向一个文件写入内容的时候，可能由于文件不存在而出现`ENOENT`错误。这个时候只需要在写入前创建这个文件就能解决。
 - 把错误扩散到客户端，如果你不知道如何处理错误，那直接抛出，上一层来处理这个异常。比如有一个负责解析`JSON`的函数，它只负责解析`JSON`所以当`JSON`的格式不正确时，它只有抛出错误，因为它并不知道业务逻辑是什么。只有让上一层来决定，上一层拿到错误后由于他知晓整个软件的业务逻辑，知道该如何处理才是正确的（比如返回对应的错误信息来告知用户：你提交的信息不正确）。
 - 重试操作，比如连接第三方网络服务出现错误，有时候重试一两次就能解决问题。重试的时候应当记录相应的日志、最多重试几次以及两次重试的间隔。同时也不要每次都重试，如果处于调用栈很深的地方，应当扩散错误，让更上层决定如何处理这种错误。
 - 只记录错误，比如用户访问了网站中一个并不存在的文件，这是很常见的错误，但是也不是什么大问题，只需要记录一下即可。

实际编写的时候，对于同步代码，使用try...catch来捕获错误。对于异步代码，根据复杂程度，你可以选择使用`EventEmitter`对象来扩散错误，或者使用回调函数来传递错误。

## 3 要不要检查函数的参数？

错误的参数最终会导致两个结果：

- 报告异常，但是可能报告可能太晚，或者调用嵌套太深，难以定位。
- 程序正常运行，但是得到错误的结果

首先我们要明确一下，为什么要检查参数？因为我们想保证程序运行正确，不然程序就会出错导致崩溃。我们调用函数的目的是为了完成自己需要的功能，除了一些恶意搞破坏的人，没人会去想：我要传个错误的参数让这个程序崩溃。
基于以上这一点我个人的观点是：

- 对于一些暴露在外的接口，以及处理用户输入的接口，尽量保证完整的参数检查。因为你没法保证每个人都正确的使用你的接口，所以为了防止程序崩溃，我们要完整的检查每个参数是否符合预期。
- 对于没有暴露在外的接口，外部人员是接触不到，只有软件的开发人员能接触到，这个时候就不一定需要参数检查了。因为我们目的本身就是完成功能没有人会故意去搞坏你的系统。另外这个度不是确定的，如果你软件本身对安全性稳定性要求比较高，那么多一些检查也是必要的。（其实这时候就不要用 JavaScript 了哈哈）
- 对于框架，因为框架本身就是为了提高开发效率、维护性和稳定性，所以为了保证用户错误使用时能够有足够的调试信息，应当尽量检查参数，在后续程序报错之前提早报错的时间，并提供详尽的错误信息。
- 不管是对外暴露的接口还是内部的接口，都要保证有规范的使用文档。保证调用者知道如何正确使用这个接口。

## 4 如何抛出错误
这个其实没多少讨论的意义，由于 node 很多函数都是使用回调，所以一般都是会将异常作为回调函数的第一个参数。以外一种就是使用 `EventEmitter`，比如`fs` 模块。使用哪个根据自己实际需求，没有特别需要遵循的原则。对于同步代码，也只能使用 `try...catch` 这种模式。

## 4.1 更详细一点

我们编写一个函数时，应当在文档中详细说明以下东西

 - 函数是做什么的
 - 需要哪些参数，包括参数的类型、参数的额外约束（比如有效的IP地址）
 - 可能会遇到的操作失败，包括他们的名称
 - 如何传递错误的，是通过 `EventEmitter` 还是回调

对于Error对象，你需要给他指定以下信息

 - `name`，错误的名称，你可以用一个简单的名称包含一个大类的错误，然后通过增加其他属性来描述更详细的信息
 - `message`，错误相关的信息，这个信息是让人阅读的。应该尽可能精炼和易读。

比如说参数错误，我们可以指定`name`为`ArgumentError`，并且在`message`中说明哪个参数错误以及正确的参数形式

如果你要把底层产生的错误传递给调用者，可以包装一下。比如说我们连接第三方网络错误时出现了错误。如果你直接把底层抛出的错误传递给调用者，那么你会得到这个`connect ECONNREFUSED`。
这样调用者即使捕获了也不清楚具体是啥错误，因此你应该考虑包装一下，在每一层添加一些关于当层有用的信息。让最终信息变成下面这个样子：

> myserver: failed to start up: failed to load configuration: failed to connect to database server: failed to connect to 127.0.0.1 port 1234: connect ECONNREFUSED。

其实这个多少也和node本身错误处理信息不足有关，看看这个[issue](https://github.com/joyent/node/issues/7005)和这个[issue](https://github.com/joyent/node/issues/7804)。

同时对于异步的方法来说，使用命名函数可以起到快速定位的作用。因为此时调用栈里保存了函数名，在打印错误信息的时候，我们可以通过函数名来显示出完整的调用顺序，能够更好的定位问题。

参考文章：
[NodeJS错误处理最佳实践](http://code.oneapm.com/nodejs/2015/04/13/nodejs-errorhandling/)
[Should a method validate its parameters](https://softwareengineering.stackexchange.com/questions/64926/should-a-method-validate-its-parameters)
[js的方法需要检查吗，检查原则是什么样的？](https://www.zhihu.com/question/56302552)
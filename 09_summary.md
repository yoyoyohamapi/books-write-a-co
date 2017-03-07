总结
=============

历经如上8个过程，我们做出了一个类似[co V2版本](https://github.com/tj/co/tree/2.0.0)的工具函数`wrap(generatorFunc)`，来帮助我们更优雅地组织异步非阻塞代码（non-blocking code），co V2的用法如下：

```js
var co = require('co');

co(function *(){
  var a = yield get('http://google.com');
  var b = yield get('http://yahoo.com');
  var c = yield get('http://cloudup.com');
  console.log(a.status);
  console.log(b.status);
  console.log(c.status);
})()

co(function *(){
  var a = get('http://google.com');
  var b = get('http://yahoo.com');
  var c = get('http://cloudup.com');
  var res = yield [a, b, c];
  console.log(res);
})()
```

目前，co已经迭代到了V4版本，这是一个建立在Promise哲学上的版本，相较于基于Thunk V2版本，该版本的用法也发生巨大的变化：

```js
co(function* () {
  var result = yield Promise.resolve(true);
  return result;
}).then(function (value) {
  console.log(value);
}, function (err) {
  console.error(err.stack);
});
```

我并没有追新，而是希望通过本文帮助大家了解如何利用ES6的generator来组织异步代码，从而优化业务逻辑。本文也是我源码分析的第二本gitbook，上一本是分析的underscore，我发现，单纯的源码分析意义不大，很容易就从一个过高的阶段入手，而渐进式的、从单一简单为题出发的，甚至不一定展示源码注释的“源码分析”可能才是最有助于读者消化的。

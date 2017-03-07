利用generator组织异步流程
==========

generator概述
--------------

[generator](https://babeljs.io/learn-es2015/#ecmascript-2015-features-generators)是ES6中[协程](https://zh.wikipedia.org/wiki/%E5%8D%8F%E7%A8%8B)的实现，用`*`标识的generator函数是其核心，generator函数
中可以容纳多个流程：

```js
function returnTwoAsync() {
    setTimeout(0, function() {
        return 2;
    });
}

function returnThreeAsync() {
    setTimeout(0, function() {
        it.next(3);
    });
}

function *gen() {
    console.log('Begin.....');
    yield 1;
    yield returnTwoAsync();
    yield returnThreeAsync();
}

const it = gen(); // 'Beign....'
it.next(); // {data:1, done:false}
it.next(); // {data:undefined, done:false}
it.next(); // {data:3, done:false}
it.next(); // {data:undefined, done:true}
```

调用generator函数，将返回一个[**迭代器（iterator）对象**](https://babeljs.io/learn-es2015/#ecmascript-2015-features-iterators-for-of)，迭代器对象具有`next()`和`throw()`方法。每次调用`next()`，就能将generator由暂停态变为执行态，并将返回一个执行信息对象，该对象包含了当前阶段的运行结果`value`，以及标识了generator是否运行完毕的`done`：

```js
{value: xxx, done: false}
```

当generator运行到`yield`关键字时，generator就会进入暂态。如果在调用`next`方法时传递了参数`next(param)`，则能将param传回generator流程，这在调用异步过程时非常有用，如上面代码片中的及`returnTwoAsync`、`returnThreeAsync`所示。

利用generator组织异步流程
-------

假设一个业务流程中穿插了多个异步过程，除了进行传统的回调解决策略，也可借助于generator进行解决，但需要做如下改造：

1. 通过generator函数来组织业务流程，可以通过`yield`来等待异步流程执行完毕，通过try-catch进行错误处理：
    ```js
    function *gen() {
        try {
            let data1 = yield callback1();
        catch(e) {
            console.error(e);
        }
        let data2 = yield callback2();
    }

    // 调用generator函数，获得操纵generator执行过程的迭代器对象
    const it = gen();
    // 首次调用`next`方法，驱动generator开始执行
    it.next()
    ```
2. 改造异步方法：
    - 遇到错误通过迭代器对象的`throw(error)`方法抛出错误，使得外部能够通过try-catch进行错误处理。
    - 如果没有发生错误，则可以继续业务流程，并利用传递参数的`next()`方法告知generator运行结果，并继续generator函数的执行。
    ```js
    function callback(err, data) {
        if(err) {
            // 抛出错误
            it.throw(err);
        } else {
            it.next(data);
        }
    }
    ```

综上，利用generator，上一节的文件大小统计代码就可以这样撰写：

```js
const fs = require('fs');

// 改造size方法
function size(filename) {
    fs.stat(filename, function(err, stat) {
        if(err) it.throw(err);
        else
            it.next(stat.size);
    });
}

function *main() {
    const sizeInfo = {
        'file1': 0,
        'file2': 0,
        'file3': 0
    };
    try {
        sizeInfo['file1'] = yield size('file1.md');
        sizeInfo['file2'] = yield size('file2.md');
        sizeInfo['file3'] = yield size('file3.md');
    catch(e) {
        console.error(e);
    }
    console.log(sizeInfo);
}

const it = main();
// 驱动iterator运行
it.next();
```

可以看到，借助于generator，我们可以用传统的阻塞式的、同步式的代码来撰写业务逻辑，也可以通过try-catch进行错误处理，但是，需要进行一定程度侵入式的改造，假设原有100个异步函数，我们就需要改造这100个函数，并且还担心这样的改造影响到原有的业务流程，因此，我们还需要进一步的处理，首先，我们会考虑创建一个运行器，来驱动我们的generator运行。

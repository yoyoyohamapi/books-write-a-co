执行器
======

上一节中，我们利用 generator，实现了以同步式代码的形式来组织异步流程，但是，如果我们有多个 generator 函数，则需要手动驱动每个 generator 的运行，而且这些代码大体是重复的：

```js
function *gen1(){}
function *gen2(){}
function *gen3(){}

let it1 = gen1();
let it2 = gen2();
let it3 = gen3();

it1.next();
it2.next();
it3.next();
```

因此，我们考虑设计一个执行器（runner），来驱动 generator 的运行：

```js
/**
 * 运行器
 * @param  {Generator} generatorFunc
 */
function runGenerator(generatorFunc) {
    // 获得generator迭代器
    let it = generatorFunc();
    next();

    /**
     * 自定义一个next函数
     */
    function next(err, res) {
        if (err) {
            it.throw(err);
        }

        const {value, done} = it.next();
        if(done) {
            return;
        }

        if(typeof value === 'function') {
            value.call(this, function(err, response) {
                if(err) {
                    next(err, null);
                } else {
                    next(null, res);
                }
            })
        }
    }
}
```

thunkify
--------

在执行器中，我们重新设计了一个 `next` 方法用于继续 generator 执行，其中的关键代码片：

```js
if(typeof value === 'function') {
    value.call(this, function(err, response) {
        if(err) {
            next(err, null);
        } else {
            next(null, res);
        }
    })
}
```

这里是为了能够以函数的形式进入暂态，使得我们的异步流程不再耦合迭代器的 `next` 方法，只需要 [thunk](https://en.wikipedia.org/wiki/Thunk) 化原有的异步函数：

```js
function async(parameters) {
    return function(cb) {
        asynFunc(parameters, function(err, data) {
            if(err)
                cb(err);
            else
                cb(null, data);
        });
    }
}
```

[node-thunkify](https://github.com/tj/node-thunkify) 是最流行的将一个函数 thunk 的工具，我也写过一个关于 thunkify 的文章：[戳我查看](http://yoyoyohamapi.me/2016/08/02/thunkify/)。借助于 thunkify，我们可以不侵入原来的异步过程：

```js
const thunkify = require('thunkify');
const fs = require('fs');

const readThunked = thunkify(fs.readFile);
```

通过运行器，第一节中的问题解决方式就变为如下：

```js
const fs = require('fs');

function size(filename) {
    // 让`size`不耦合`next()`
    return function(fn) {
        fs.stat(filename, function(err, stat) {
            if(err) fn(err);
            else
                fn(null, stat.size);
        });
    }
}

function runGenerator(gen) {
    // 先获得迭代器
    const it = gen();
    // 驱动generator运行
    next();

    function next(err, res) {
        if(err) {
            return it.throw(err);
        }

        const { value, done } = it.next(res);
        if(done) {
            return;
        }

        if(typeof value === 'function') {
            value.call(this, function(err, res) {
                if(err) {
                    next(err, null);
                } else {
                    next(null, res);
                }
            });
        }

    }
}

function *main() {
    const sizeInfo = {
        'file1': 0,
        'file2': 0,
        'file3': 0
    };
    try{
        sizeInfo['file1'] = yield size('file1.md');
        sizeInfo['file2'] = yield size('file2.md');
        sizeInfo['file3'] = yield size('file3.md');
    } catch(error) {
        console.error('error:', error);
    }
    console.dir(sizeInfo)
}

runGenerator(main);
```

回到问题本身，对于三个文件信息的统计，我们实际上可以并行处理，而不需要逐个统计、逐个等待，这才能利用到异步编程的优势，亦即，我们在利用了传统的同步式风格组织代码后，还希望执行保持异步编程的优势。

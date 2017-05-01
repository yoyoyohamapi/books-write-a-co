俘获返回值
==========

假定我们不需要直接打印文件信息，而是希望通过传统 `return` 来返回文件信息，这样，有助于解除信息打印和信息获取间的耦合，让 `*main` 只关注获得文件信息，而可以把信息的打印，再处理等工作交给其他函数完成：

```js
function *main() {
    const sizeInfo = {
        'file1': 0,
        'file2': 0,
        'file3': 0
    };

    sizes = yield[
        size('file1.md'),
        size('file2.md'),
        size('file3.md')
    ];

    sizeInfo['file1'] = sizes[0];
    sizeInfo['file2'] = sizes[1];
    sizeInfo['file3'] = sizes[2];
    return sizeInfo;
}
```

但是，接下来我们就手足无措了，如下代码是肯定拿不到文件信息的，因为 `*main` 方法尚没有渠道把 `sizeInfo` 交付给运行器：

```js
let sizeInfo = runGenerator(main);
console.dir(sizeInfo); // undefined
```

由于业务流程是异步的，所以，想要 generator 把最终获得值递交给执行器，也只有通过异步的方式传递，下面的调用方式才有可能获得 `sizeInfo`：

```js
runGenerator(main, function(err, sizeInfo){
    if(err) {
        console.error('error', err);
    } else {
        console.dir(sizeInfo);
    }
});
```

为了实现上述的调用方式，改造我们的 `runGenerator`：

```js
function runGenerator(gen, cb) {
    // 先获得迭代器
    const it = gen();
    // 驱动generator运行
    next();

    function next(err, res) {
        if (err) {
            try {
                // 防止报错：Unhandled promise rejection
                return it.throw(err);
            } catch (e) {
                return cb(err);
            }
        }

        const { value, done } = it.next(res);
        if (done) {
            cb(null, value);
        }
        thunk = toThunk(value);
        if (typeof thunk === 'function') {
            thunk.call(this, function (err, res) {
                if (err) {
                    next(err, null);
                } else {
                    next(null, res);
                }
            });
        }
    }
}
```

这样做还不是最好的，我们可以把运行器也 thunk 化，这样，能将对 generator 函数的封装和驱动 generator 运行分开：

```js
function wrap(gen) {
    // 先获得迭代器
    const it = gen();

    return function (cb) {
        // 驱动generator运行
        next();

        function next(err, res) {
            if (err) {
                try {
                    // 防止报错：Unhandled promise rejection
                    return it.throw(err);
                } catch (e) {
                    return cb(err);
                }
            }

            const { value, done } = it.next(res);
            if(done) {
                cb(null, value);
            }
            thunk = toThunk(value);
            if(typeof thunk === 'function') {
                thunk.call(this, function(err, res) {
                    if(err) {
                        next(err, null);
                    } else {
                        next(null, res);
                    }
                });
            }
        }
    }
}
```

测试一下：

```js
// 封装
let wrapped = wrap(main);
function print(err, sizeInfo) {
    if(err) {
        console.error(err);
    } else {
        console.dir(sizeInfo);        
    }
}
// 运行
wrapped(print);
// { file1: 5384, file2: 2712, file3: 13942 }
```

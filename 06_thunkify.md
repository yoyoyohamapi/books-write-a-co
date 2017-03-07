thunk化
=============

在我们的`runGenerator`中，有如下代码片：
```js
function runGenerator(gen) {
    // ...
    if (isPromise(value)) {
        value.then(function(res) {
            next(null, res);
        }, function(err) {
            next(err);
        });
    }
    if (Array.isArray(value)) {
        // 存放异步任务结果
        const results = [];
        // 等待完成的任务数
        let pending = value.length;
        value.forEach(function (func, index) {
            func.call(this, function (err, res) {
                if (err) {
                    next(err);
                } else {
                    // 保证结果的存放顺序
                    results[index] = res;
                    // 直到所有任务执行完毕
                    if (--pending === 0) {
                        next(null, results);
                    }
                }
            })
        })
    }
    if (typeof value === 'function') {
        value.call(this, function (err, res) {
            if (err) {
                next(err, null);
            } else {
                next(null, res);
            }
        });
    }
}
```

这一部分代码的重复率很高，都是：
    - 出错时：执行`next(err)`
    - 正确时：执行`next(null, res)`

如果我们首先对`value`thunk化：

```js
function toThunk(fn) {
    if(Array.isArray(fn)) {
        const results = [];
        // 等待完成的任务
        let pending = fn.length;
        return function(cb) {
            let finished = false;
            fn.forEach(function(func, index) {
                if(finished) {
                    return;
                }
                func = toThunk(func);
                func.call(this, function(err, res) {
                    if(err) {
                        finished = true;
                        cb(err);
                    } else {
                        results[index] = res;
                        // 如果再无任务，才返回`results`
                        if(--pending === 0) {
                            cb(null, results);
                        }
                    }
                } );
            })
        }
    }else if(isPromise(fn)) {
        return function(cb) {
            return fn.then(function(res){
                cb(null, res);
            }, function(err){
                cb(err);
            })
        }
    }
}
```

那么就能避免上文的重复问题，现在，`runGenerator`中只需要一个判断逻辑了：

```js
function runGenerator(gen) {
    // 先获得迭代器
    const it = gen();
    // 驱动generator运行
    next();

    function next(err, res) {
        if (err) {
            return it.throw(err);
        }

        const { value, done } = it.next(res);
        if (done) {
            return;
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

并行处理
========

传统回调的解决方式
------------------

对于统计三个文件信息，我们想要通过并行处理提升性能，使用传统的回调解决策略，会是这样的：

```js
let info = {};

function print() {
    keys = Object.keys(info);
    if (keys.length === 3 ) {
        console.dir(info);
    }
}

fs.stat('file1.md', function(err, stat) {
    if(!err) {
        info['file1'] = stat.size;
        print();
    }
});

fs.stat('file2.md', function(err, stat) {
    if(!err) {
        info['file2'] = stat.size;
        print();
    }
});

fs.stat('file3.md', function(err, stat) {
    if(!err) {
        info['file3'] = stat.size;
        print();
    }    
})
```

传统回调组织并行任务时，乍看起来还好，但是我们发现，其免不了的是需要对任务完成情况（即并行任务的是否结束）进行重复性判断，比如上例中，我们要判断三个文件是否都统计完毕，就需要在各个异步任务中重复执行 `print` 方法中的判断逻辑：

```js
if (keys.length === 3 ) {
    console.dir(info);
}
```

利用 generator 并行处理任务
---------------------------

除了重复，上述判断逻辑无疑也是强侵入性的，而如果使用 generator，我们希望最终能通过一个数组就组织并行任务：

```js
function *main(){
    let sizeInfo = {};
    let sizes = yield [
        size('file1.md'),
        size('file2.md'),
        size('file3.md')
    ];
    sizeInfo['file1'] = sizes[0];
    sizeInfo['file2'] = sizes[1];
    sizeInfo['file3'] = sizes[2];
}
```

实现这个目标之前，需要考虑如下几点：

-	异步任务结果的组织顺序能与声明顺序保持一致，而不是与完成顺序一致，因为任务的完成顺序是不定的
-	并行任务中的任意一个失败，不再继续后续任务，而是抛出错误，交给开发者俘获处理

为此，我们需要考虑修改运行器，当 generator 遇到数组进入暂态时，额外进行处理：

```js
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

        if (Array.isArray(value)) {
            // 存放异步任务结果
            const results = [];
            // 等待完成的任务数
            let pending = value.length;
            // 当任务队列里任意一个任务发生错误时，终止所有任务的继续
            let finished = false;
            value.forEach(function(func, index) {
                func.call(this, function(err, res) {
                    if(err) {
                        finished = true;
                        next(err);
                    } else {
                        // 保证结果的存放顺序
                        results[index] = res;
                        // 直到所有任务执行完毕
                        if(--pending === 0) {
                            next(null, results);
                        }
                    }
                })
            })
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
```

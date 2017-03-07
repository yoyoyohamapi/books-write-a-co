支持Promise
==============
Promise是解决传统回调陷阱（callback hell）的一个策略，如果还不了解的Promise的，这篇[Javascript-Promise迷你书](http://liubin.org/promises-book/)会是最好选择，详细介绍了前世今生、设计哲学和使用方法。

如果使用promise，我们解决第一节中的问题就会是如下代码：

```js
function size(filename) {
    return new Promise((resolve, reject)=>{
        fs.stat(filename, (err, stat) => {
            if(err)
                reject(err);
            else
                resolve(stat.size);
        });
    });
}

function *main() {
    const sizeInfo = {
        'file1': 0,
        'file2': 0,
        'file3': 0
    };
    try{
        sizes = yield Promise.all([
            size('file1.md'),
            size('file2.md'),
            size('file3.md')
        ]);

        sizeInfo['file1'] = sizes[0];
        sizeInfo['file2'] = sizes[1];
        sizeInfo['file3'] = sizes[2];
    } catch(error) {
        console.error('error:', error);
    }
    console.dir(sizeInfo);
}
```

可以用如下的方式判断一个对象是否是Promise对象：

```js
function isPromise(obj) {
    return obj && typeof obj.then === 'function';
}
```

为了支持以Promise构建业务逻辑，就需要对运行器作出如下修改：

```js
function runGenerator(gen) {
    // 先获得迭代器
    const it = gen();
    // 驱动generator运行
    next();

    function next(err, res) {
        // ...
        if (isPromise(value)) {
            value.then(function(res) {
                next(null, res);
            }, function(err) {
                next(err);
            });
        }
        if (Array.isArray(value)) {
            // ...
        }
        if (typeof value === 'function') {
            // ...
        }

    }
}
```

到本节末为止，我们借助于一个运行器`runGenerator`就可以用更优雅的、同步式的方式来组织我们的业务流程，在接下来的章节，我们着重优化这个工具函数。

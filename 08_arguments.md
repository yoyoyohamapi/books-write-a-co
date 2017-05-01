向generator函数传递参数
=======================

我们修改一下第一节提出的问题，现在，不再是固定统计 `file1.md`，`file2.md`，`file3.md` 三个文件的大小，而是支持任意传递任意的文件序列进行统计，显然，我们就要允许 generator 函数允许接收参数：

```js
function *main(files) {
    // 初始化信息
    const sizeInfo = files.reduce((info, file) => {
        info[file] = 0;
        return info;
    }, {});

    try{
        const requests = files.map((file) => {
            return size(file);
        });

        sizes = yield requests;

        sizes.forEach((size, index) => {
            sizeInfo[files[index]] = sizes[index];
        });

        return sizeInfo;
    } catch(error) {
        console.error('error:', error);
    }
}
```

当我们通过`wrap`包装generator函数后，希望能以如下方式调用包装后的函数，驱动generator的运行：

```js
wrap(main)(['file1.md', 'file2.md', 'file3.md'], function(err, sizeInfo){
    if(err) {
        console.error(err);
    } else {
        console.dir(sizeInfo);
    }
});
```

在这种调用方式中，我们规范最后一个参数为固定的回调函数，其余皆为传递给 generator 函数的参数，为此，我们需要修改 `wrap` 包装器，让其在调用 generator 函数时，将参数交付：

```js
function wrap(gen) {
    return function (cb) {
        const args = Array.prototype.slice.call(arguments);
        const length = args.length;
        // 判断最后一个参数是否为回调函数
        if(length && 'function' === typeof args[length-1]) {
            cb = args.pop();
            it = gen.apply(this, args);
        } else {
            return;
        }
        // 驱动generator运行
        next();

        function next(err, res) {
            if(err) {
                return it.throw(err);
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

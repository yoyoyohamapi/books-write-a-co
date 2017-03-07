# 传统的回调

现在，我们有三个markdown文件`file1.md`,`file2.md`,`file3.md`，我们想要统计这三个文件的大小信息，并输出为以下格式：

```js
{ file1: 5384, file2: 2712, file3: 13942 }
```

在node中，我们知道[`fs.stat`](https://nodejs.org/dist/latest-v6.x/docs/api/fs.html#fs_fs_stat_path_callback)这个异步API可以用来获得文件信息，该方法接受两个参数：（1）`path`：文件路径；（2）callback：一个[error-first](http://fredkschott.com/post/2014/03/understanding-error-first-callbacks-in-node-js/)型的回调函数，因此，使用最**传统**的回调方法，我们处理如上问题的代码如下：

```js
const fs = require('fs');

function error(err) {
    throw new Error(err);
}

function main() {
    const sizeInfo = {
        'file1': 0,
        'file2': 0,
        'file3': 0
    };
    fs.stat('file1.md', function(err, stat) {
        if(err) return error(err);
        sizeInfo['file1'] = stat.size;
        fs.stat('file2.md', function(err, stat) {
            if(err) return error(err);
            sizeInfo['file2'] = stat.size;
            fs.stat('file3.md', function(err, stat){
                if(err) return error(error);
                sizeInfo['file3'] = stat.size;
                console.dir(sizeInfo);
            });
        })
    });
}

main();
```

因为是三个文件，我们用传统的回调方式写出了三层的嵌套，整个代码呈现**`>`**型，形成了[回调地狱（callback hell）](http://callbackhell.com/)，如果显示器尺寸不够，甚至需要拖动滚动条才能阅览整个代码，十分影响编码时的快感。其次，对于熟悉java，php，或者C++语言的开发者，不能用try-catch进行错误处理，而需要撰写无数的`if-else`来处理异常或者错误，就更是蛋疼了。

为了从深陷的回调地狱中爬出来，下一节中，我们将使用ES6提供的generator来优化异步逻辑。

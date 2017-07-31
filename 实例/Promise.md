使用Promise可以表达一连串嵌套的异步操作。

每个Promise实例代表一个异步操作。

~~~
var p = new Promise(function (resolve) {
    setTimeout(function () {
        resolve({});
    }, 100);
});

p.then(function (data) {

});
~~~

参数resolve将接受一个函数，该函数用于异步回调时传递结果。
然后Promise的实例使用then函数，便可以处理resolve传递过来的结果。

then可以链式操作，增强Promise的表达能力。每次执行then部署的任务``相当于抛出一个新的Promise实例。
因此可以优雅地表达在异步操作中抛出异步操作。

~~~
p.then(function (data) {
    //p返回结果后执行
    return new Promise(p1);
}).then(function(data){
    //p1返回结果后执行
});
~~~

Promise有一套实现标准和使用方法，具体此处不再详述。下文用ES代码实现Promise这个轮子。

# 实现的范围

Promise对象：

* Promise.resolve(value)//接受一个异步结果，返回一个成功的Promise实例。
* Promise.reject(reason)//接受一个异步错误，返回一个失败的Promise实例。

Promise实例：

* Promise.prototype.then(onSuccess,onFailure)//接受两个参数，配置异步成功和失败时的处理函数，返回一个新的Promise实例。
* Promise.prototype.catch(onFailure)//接受一个参数，配置异步失败时的处理函数，返回一个新的Promise实例。

# 代码骨架

~~~
(function () {
    "use strict";
    //状态选项
    var emStatus = {
        pending: 0,//进行中。
        resolved: 1,//成功。
        rejected: 2//失败。
    };

    var Promise = function (task) {

        this.then = function (onSuccess, onFailure) {

        };
        this.catch = function (onFailure) {

        };

        var status = emStatus.pending,//异步（同步）操作的状态
            //成功时使用的对象
            Success = {
                value,
                event,
                run: function () {

                }
            },
            //失败时使用的对象
            Faliure = {
                reason,
                event,
                run: function () {

                }
            },
            //将消息通知给then部署的任务。
            Chain = {
                resolve,
                reject
            };

        var resolve = function (value) {

        };

        var reject = function (reason) {

        };
        //给异步（同步）操作注入resolve和reject
        task(resolve, reject);
    };

    Promise.resolve = function (value) {
        return new Promise(function (resolve) {
            resolve(value);
        });
    };

    Promise.reject = function (reason) {
        return new Promise(function (resolve, reject) {
            reject(reason);
        });
    };

    return Promise;
})();
~~~
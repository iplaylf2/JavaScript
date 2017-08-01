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
    //状态枚举
    var emStatus = { pending: 0, resolved: 1, rejected: 2 };

    var Promise = function (task) {

        this.then = function (onSuccess, onFailure) {

        };
        this.catch = function (onFailure) {

        };

        var status = emStatus.pending,//任务的状态
            boxValue,//任务的结果
            chain = { resolve, reject };//抛出的Promise实例的触发装置

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

then和catch所部署的函数的触发，取决于Promise实例的内部状态。所以then和catch写进了Promise实例的构造函数，以引用包含Promise实例内部状态的环境。
如此一来，Promise实例只能被创建时注入的resolve和reject函数所触发。这种具有内部封闭性的对象才能称得上是真正的对象。

Promise实例我设置了三个状态变量。

* status为任务的状态，有三个，未开始（pending），成功（resolved）和失败（rejected）。
* boxValue为任务的结果，成功失败的结果都会被保存。
* 因为then会抛出一个新的Promise实例，所以需要保存该实例的触发装置，使用chain保存resolve和reject触发装置。

Promise.resolve/Promise.reject如意义上的抛出一个新的成功/失败Promise实例。

# 骨架的优化和改进。

catch函数实际上相当于then(null,onFailure)。因此不用每次在构造函数中构造，只需要使用原型链的方式访问。

同一个异步Promise实例的then函数可以被触发多次，因此chain应该是个数组。

~~~
(function () {
    "use strict";

    var emStatus = { pending: 0, resolved: 1, rejected: 2 };

    var Promise = function (task) {

        this.then = function (onSuccess, onFailure) {

        };

        var status = emStatus.pending,
            boxValue,
            chainArr = [];

        var resolve = function (value) {

        };

        var reject = function (reason) {

        };
        task(resolve, reject);
    };
    Promise.prototype = {
        catch: function (onFailure) {
            return this.then(null, onFailure);
        }
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

此时只要实现then，resolve和reject。

# then

Promise实例使用then的时候，他可能有三种状态。仍未触发回调的pending，已经有结果的resolved和rejected。

~~~
this.then = function (onSuccess, onFailure) {
    switch (status) {
        case emStatus.pending:
            return new Promise(function (resolve, reject) {
                var chain = {};
                if (onSuccess) {
                }
                else {
                    chain.resolve = resolve;
                }
                if (onFailure) {
                }
                else {
                    chain.reject = reject;
                }
                chainArr.push(chain);
            });
        case emStatus.resolved:
            if (onSuccess) {
            }
            else {
                return Promise.resolve(boxValue);
            }
        case emStatus.rejected:
            if (onFailure) {
            }
            else {
                return Promise.reject(boxValue);
            }
    }
};
~~~

## pending

如果then没有传入参数，即没有部署操作结果的处理函数。那么返回的Promise实例将继承之前操作的结果。

如果pending状态时部署了处理函数，那么在未来，将会用该函数对操作返回结果进行处理。被处理后的结果，将是一个resolved的结果。期间发生异常，将被判断为rejected。

这个处理函数包括操作成功和失败时的处理函数：

~~~
var result;
try {
    result = on(value);
}
catch (e) {
    reject(e);
    return;
}
if (result instanceof Promise) {
    result.then(function (value) {
        resolve(value);
    });
}
else {
    resolve(result);
}
~~~

如果处理函数接着抛出一个Promise实例，那么需要等该Promise实例触发回调后才把处理函数视为返回了结果。

以上就是处理函数的内容，那么我们需要再将其抽象为下一个Promise的触发装置。
而且，因为处理函数包括成功和失败的处理函数，所以可以抽象一个通用的模板。我给他取名nextPromise。

~~~
var nextPromise = function (on, resolve, reject) {
    return function (value) {
        var result;
        try {
            result = on(value);
        }
        catch (e) {
            reject(e);
            return;
        }
        if (result instanceof Promise) {
            result.then(function (value) {
                resolve(value);
            });
        }
        else {
            resolve(result);
        }
    };
};
~~~

使用该模板，就能生成下一个Promise的触发装置。

~~~
chain.resolve = nextPromise(onSuccess, resolve, reject);
~~~

## resolved和rejected

Promise实例完成状态的then，情况比较简单。只要保证返回结果一定是Promise就可以了，我给他取名getPromise。

~~~
var getPromise = function (on, value) {
    var result;
    try {
        result = on(value);
    }
    catch (e) {
        return Promise.reject(e);
    }
    if (result instanceof Promise) {
        return result;
    }
    else {
        return Promise.resolve(result);
    }
};
~~~

## then

所以then的最终代码是

~~~
this.then = function (onSuccess, onFailure) {
    switch (status) {
        case emStatus.pending:
            return new Promise(function (resolve, reject) {
                var chain = {};
                if (onSuccess) {
                    chain.resolve = nextPromise(onSuccess, resolve, reject);
                }
                else {
                    chain.resolve = resolve;
                }
                if (onFailure) {
                    chain.reject = nextPromise(onFailure, resolve, reject);
                }
                else {
                    chain.reject = reject;
                }
                chainArr.push(chain);
            });
        case emStatus.resolved:
            if (onSuccess) {
                return getPromise(onSuccess, boxValue);
            }
            else {
                return Promise.resolve(boxValue);
            }
        case emStatus.rejected:
            if (onFailure) {
                return getPromise(onFailure, boxValue);
            }
            else {
                return Promise.reject(boxValue);
            }
    }
};
~~~

# resolve和reject

一个Promise实例的状态只会被改变一次，再要改变的话就抛出异常吧。
改变Promise实例的状态后，就是触发之前所有使用then抛出的新Promise实例。

~~~
var resolve = function (value) {
    if (status === emStatus.pending) {
        status = emStatus.resolved;
        boxValue = value;

        chainArr.forEach(function (chain) {
            chain.resolve(value);
        });
    }
    else {
        throw value;
    }
};
~~~

实际上，reject跟resolve做同一样的事情。

~~~
var reject = function (reason) {
    if (status === emStatus.pending) {
        status = emStatus.rejected;
        boxValue = reason;

        chainArr.forEach(function (chain) {
            chain.reject(reason);
        });
    }
    else {
        throw reason;
    }
};
~~~

# 最终代码

~~~
(function () {
    "use strict";
    var emStatus = { pending: 0, resolved: 1, rejected: 2 };

    var Promise = function (task) {
        this.then = function (onSuccess, onFailure) {
            switch (status) {
                case emStatus.pending:
                    return new Promise(function (resolve, reject) {
                        var chain = {};
                        if (onSuccess) {
                            chain.resolve = nextPromise(onSuccess, resolve, reject);
                        }
                        else {
                            chain.resolve = resolve;
                        }
                        if (onFailure) {
                            chain.reject = nextPromise(onFailure, resolve, reject);
                        }
                        else {
                            chain.reject = reject;
                        }
                        chainArr.push(chain);
                    });
                case emStatus.resolved:
                    if (onSuccess) {
                        return getPromise(onSuccess, boxValue);
                    }
                    else {
                        return Promise.resolve(boxValue);
                    }
                case emStatus.rejected:
                    if (onFailure) {
                        return getPromise(onFailure, boxValue);
                    }
                    else {
                        return Promise.reject(boxValue);
                    }
            }
        };

        var status = emStatus.pending,
            boxValue,
            chainArr = [];

        var resolve = function (value) {
            if (status === emStatus.pending) {
                status = emStatus.resolved;
                boxValue = value;

                chainArr.forEach(function (chain) {
                    chain.resolve(value);
                });
            }
            else {
                throw value;
            }
        };
        var reject = function (reason) {
            if (status === emStatus.pending) {
                status = emStatus.rejected;
                boxValue = reason;

                chainArr.forEach(function (chain) {
                    chain.reject(reason);
                });
            }
            else {
                throw reason;
            }
        };

        try {
            task(resolve, reject);
        }
        catch (e) {
            reject(e);
        }
    };
    Promise.prototype = {
        catch: function (onFailure) {
            return this.then(null, onFailure);
        }
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

    var nextPromise = function (on, resolve, reject) {
        return function (value) {
            var result;
            try {
                result = on(value);
            }
            catch (e) {
                reject(e);
                return;
            }
            if (result instanceof Promise) {
                result.then(function (value) {
                    resolve(value);
                });
            }
            else {
                resolve(result);
            }
        };
    };
    var getPromise = function (on, value) {
        var result;
        try {
            result = on(value);
        }
        catch (e) {
            return Promise.reject(e);
        }
        if (result instanceof Promise) {
            return result;
        }
        else {
            return Promise.resolve(result);
        }
    };

    return Promise;
})();
~~~
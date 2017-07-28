eval函数可以主动把字符串加入当前环境的解释程序中。

直接效果就是执行了一段文本形式的代码。

~~~
var foo = 20;

var func = function () {
    var foo = 33;
    eval("var result=foo;foo=foo+1;");
    console.dir(result);
};

func();//33
console.dir(foo);//20
~~~


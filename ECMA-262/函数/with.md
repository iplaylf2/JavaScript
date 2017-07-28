with关键字可以产生一个特殊的环境，访问该环境的时候与访问对象一样。

~~~
var obj = { a: 0 };

var foo = {};
with (obj) {
    
    foo.change = function () {
        a = a + 1;
    };

    foo.look = function () {
        console.dir(a);
    };

}

foo.look();//0
foo.change();
foo.look();//1

console.dir(obj.a);//1

obj = null;

foo.look();//1
foo.change();
foo.look();//2
~~~

在with(obj)产生的环境中访问obj的属性，如同用obj本身访问属性。

甚至能访问原型链。

~~~
var obj = {
    b: function () {
        return this;
    }
};

var foo;
with (obj) {
    foo = function () {
        console.dir(b());
        console.dir(toString());
    }
}

foo();
~~~

![](../../images/TIM截图20170728092852.jpg)
---
title: '谈谈setTimeout的作用域以及this的指向问题'
date: 2019-09-16 13:03:44
tags:
copyright:
---

setTimeout的常见用法是让某个方法延迟执行。我们知道，setTimeout方法是挂在window对象下的。《JavaScript高级程序设计》第二版中，写到：“超时调用的代码都是在全局作用域中执行的，因此函数中this的值在非严格模式下指向window对象，在严格模式下是undefined”。在这里，我们只讨论非严格模式。

setTimeout接受两个参数，第一个是要执行的代码或函数，第二个是延迟的时间。

一、先说结论：setTimeout中所执行函数中的this，永远指向window！！注意是要**延迟执行的函数中的this**哦！！

1. 直接使用，代码1.1：

```javascript
 setTimeout("alert(this)", 1);// [object Window]
```

2. 在一个对象中调用setTimeout试试，代码1.2：

```javascript
var obj = {
        say: function () {
            setTimeout("alert('in obj ' + this)", 0)
        }
    }
    obj.say();   // in obj [object Window]
```

3. 将执行的代码换成匿名函数试试，代码1.3：

```javascript
varobj = {
        say: function () {
            setTimeout(function () {
                alert(this)
            }, 0)
        }
    }
    obj.say();   //  [object Window]
```

4. 换成函数引用再试试吧，代码1.4：

```javascript
function talk() {
  alert(this);
}
var obj = {
  say: function() {
    setTimeout(talk, 0)
  }
}
obj.say();   //  [object Window]
```

恩，貌似得到的结论是正确的，setTimeout中的延迟执行函数中的this指向了window。这里我反复的强调，是延迟执行函数中的this，是因为，我们经常会面对两个this。**一个是setTimeout调用环境中的this，一个就是延迟执行函数中的this**。这两个this有时候是不同的。有些不放心？？再多写一些代码测试一下！　　

 

 二、setTimeout中的两个this到底指向谁？？为了便于区分，**我们把setTimeout调用环境下的this称之为第一个this，把延迟执行函数中的this称之为第二个this**，并在代码注释中标出来，方便您区分。先说得出的结论：**第一个this的指向是需要根据上下文来确定的，默认为window；第二个this就是指向window**。然后我们通过代码来验证下。

1. 函数作为方法调用还是构造函数调用，this是不同的。先看代码，代码2.1：

```javascript
function Foo() {
    this.value = 42;
    this.method = function() {
        // this 指向全局对象
        alert(this)   // 输出window  第二个this
        alert(this.value); // 输出：undefined   第二个this
    };
    setTimeout(this.method, 500);  // this指向Foo的实例对象  第一个this
}
new Foo();
```

我们new了一个Foo对象，那么this.method中的this指向的是new的对象，否则无法调用method方法。但是进了method方法后，方法中的this又指向了window，因此this.value的值为undefined。

我们在外层添加一段代码，再看看，代码2.2：

```javascript
var value=33;
 
function Foo() {
    this.value = 42;
    this.method = function() {
        // this 指向全局对象
        alert(this)   // 输出window    第二个this
        alert(this.value); // 输出：33   第二个this
    };
    setTimeout(this.method, 500);  // 这里的this指向Foo的实例对象  第一个this
}
new Foo();
```

从这里，可以明显的看到，method方法中的this指向的是window，因为可以输出外层的value值。那为什么setTimeout中的this指向的是Foo的实例对象呢？

我觉得代码2.2就等价于下面的代码，如代码2.3：

```javascript
var value=33;
 
function Foo() {
    this.value = 42;
    setTimeout(function(){alert(this);alert(this.value)}, 500);  // 先后输出 window   33  这里是第二个this
}
new Foo();
```

setTimeout中的第一个参数就是一个单纯的函数的引用而已，而函数中的this仍然指向的是window。在setTimeout(this.method, time) 中的this是可以根据上下文而改变的，其最终的目的是要得到一个函数指针。我们再来验证一下，看代码2.4:

```javascript
function method() {
  alert(this.value);  // 输出 42  第二个this
}
 
function Foo() {
    this.value = 42;
    setTimeout(this.method, 500);  // 这里this指向window   第一个this
}
 
Foo();
```

这次我们将Foo当成方法直接执行，method方法放到外层，即挂在window上面。而this则指向了window，因此可以调用method方法。method方法中的this仍然指向window，而Foo()执行的时候，对window.value进行了赋值(this.value=42)，因此输出了42。

　

三、实践。知道了得出的结论，我们来阅读一下比较奇葩的一些代码，进行验证。　　

首先在一个函数中，调用setTimeout。代码3.1：

```javascript
var test = "in the window";
 
setTimeout(function() {alert('outer ' + test)}, 0); // 输出 outer in the window ，默认在window的全局作用域下
 
function f() {
  var test = 'in the f!';  // 局部变量，window作用域不可访问
  setTimeout('alert("inner " + test)', 0);  // 输出 outer in the window, 虽然在f方法的中调用，但执行代码(字符串形式的代码)默认在window全局作用域下，test也指向全局的test
}
 
f();
```

在f方法中，setTimeout中的test的值是外层的test，而不是f作用域中的test。再看代码3.2：

```javascript
var test = "in the window";
 
setTimeout(function() {alert('outer' + test)}, 0); // outer in the window  ，没有问题，在全局下调用，访问全局中的test
 
function f() {
  var test = 'in the f!';
  setTimeout(function(){alert('inner '+ test)}, 0);  // inner in the f! 有问题，不是说好了执行函数中的this指向的是window吗？那test也应该对应window下的值才对，怎么test的值却是 f()中的值呢？？？？
}
 
f();
```

按照前面的经验，f中的setTimeout中的test也应该明明应该是指向外层的test才对吧？？？我们注意到，这个f里面的setTimeout中的第一个参数是一个匿名函数，这是上面两端代码最大的不同。而只要是函数就有它的作用域，我们可以将上面的代码替换成下面的代码3.3：

```javascript
var test = "in the window";
 
setTimeout(function() {alert('outer ' + test)}, 0); // in the window
 
function f() {
  var test = 'in the f!';
 
  function ff() {alert('inner ' + test)} // 能访问到f中的test局部变量
 
  setTimeout(ff, 0);  // inner in the f!
}
 
f();
```

 再看一段更清晰的代码，3.4：

```javascript
var value=33;
 
function Foo() {
    var value = 42;
    setTimeout(function(){alert(value);alert(this.value)}, 500);  // 先后输出 42 然后输出33  这里的this是第二个this
}
new Foo();
```

可以确定，延迟执行函数中的this的确是指向了window，毫无疑问，上面的所有代码都可以验证哈。但是延迟执行函数中的其他变量需要根据上下文来确认。

修改代码3.4为3.5，去掉匿名函数的调用方式，会更加清晰：

```javascript
var value=33;
 
function Foo() {
    var value = 42;
    function ff() {
      alert(value);  // 42
      alert(this.value);  // 33
    }
    setTimeout(ff, 500);  // 先后输出 42   33 
}
Foo(); // 直接执行，跟普通函数没有区别
```

因此，如果去掉Foo中的value=42的话，那么value的值等于多少呢？undefined还是外层的33？？请看3.5：

```javascript
var value=33;
 
function Foo() {
    function ff() {
      alert(value);   // 输出33
      alert(this.value);  // 输出33  this指向window
    }
    setTimeout(ff, 500);  // 先后输出 33  33
}
Foo();
```

没错，就是外层的33，因为ff可以访问到window下的value值，就如同setTimeout中的匿名函数一样。　　　　

最后，我们通过对象的方式进行调用，代码3.6：

```javascript
var obj = {
  name: 'hutaoer',
  say: function() {
    var self = this;
    setTimeout(function(){
      alert(self);   // 输出 object ，指向obj
      alert(this);   // 第二个this，指向window，我心永恒，从未改变
      alert(self.name)  // 输出 hutaoer
    }, 0)
  }
}
 
obj.say();
```

 

最后，如果您到看懂了上面的例子，那么我们可以回顾一下得出的一些结论咯：

一、**setTimeout中的延迟执行代码中的this永远都指向window**

二、**setTimeout(this.method, time)这种形式中的this，即上文中提到的第一个this，是根据上下文来判断的，默认为全局作用域，但不一定总是处于全局下，具体问题具体分析。**

三、**setTimeout(匿名函数, time)这种形式下，匿名函数中的变量也需要根据上下文来判断，具体问题具体分析**。
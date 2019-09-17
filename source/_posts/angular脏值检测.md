---
title: 'angular脏值检测'
date: 2019-09-16 23:16:53
tags: 脏值检测
categories: angular
copyright: true
---

# 一: 概念简介

 脏值检测，简单的说就是在MVC的构架中，视图会通过模型的change事件来更新自己。

脏值检测的核心代码是观察者模式的实现，其机制会执行digest循环，在特定UI组件的上下文执行注册的各种表达式。

那么脏值检测在单页应用扮演了什么角色呢？

 ![img](https://images2015.cnblogs.com/blog/1028445/201707/1028445-20170708152854378-1088268803.png)
为了支持单页SPA应用，angular1引入了指令的概念，能够扩展HTML标签并且封装相关的DOM逻辑，以此来构建组件，组件再组合成一个个网页。angular2也保留了指令的概念。那么angular2和1版本的区别在哪里呢？angular2的指令则是简化版的指令API，能通过属性型的指令给DOM标签添加行为。

而与此同时，组件则可以看做指令API的补充，既可以添加模板，也可以添加行为，组件继承自指令。

```
@Directive({
  selector: '[saTooltip]'
})
export class Tooltip {
 //略
}
```

# 二: angular1中的脏值检测


```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <script src="angular.js"></script>
    <!--引入版本为1.5.0-->
    <script>
        angular.module("app",[])
            .controller("mainCtrl",function($scope){
            $scope.label="hello world!";
        });
    </script>
</head>
<body ng-app="app" ng-controller="mainCtrl">

     <div>
         <span>label值:{{label}}</span>
         <br/>
         <input type="text" ng-model="label"/>
     </div>
</body>
</html>
```

浏览器结果

![img](https://images2015.cnblogs.com/blog/1028445/201707/1028445-20170708153549612-1012918620.png)

angular1的数据绑定，酷炫而神秘！可是它幕后实际发生了什么呢？

(1)首先，在指令ng-model和双大括号(实际上是ng-bind指令)内部绑定了多个watcher(监视器)。

(2)接下来，当指定的事件发生之后，angular循环遍历所有的监视器，并执行对应scope上下文的表达式。这里就是digest循环。

(3)最后，比对两次结果，如果不相同就调用回调函数。

但这种发生特定事件(有可能在框架管理范围内，也可能在范围外部)会调用digest循环有缺点。比如回调函数有setTimeOut定时器，定时器把绑定到Scope上的属性给修改了，那么angular就无法察觉对象的改变了。angular2解决这个问题。

# 三: angular2更优的脏值检测

1.zone.js库

zone.js是angular2的一个库，可以在javascript实现各种分区。分区代表一个执行上下文。angular2利用了zone.js来拦截浏览器中的各种异步事件，然后在正确的时机调用digest循环，完全消除了需要angular开发者显式调用digest循环的情况。

2.单向数据流

在第一章的例子里，label表达式在ng-model指令下发生改变，也会通知label的双大括号表达式改变值，这隐含了指令之间互相影响、有依赖关系。跨监视器的依赖会创建出各种纠缠不清的额数据流，导致最后可能很难追踪，进而导致难以预料的错误。

angular2保留了脏值检测的概念用来检测数据，但强制使用了单向数据流。

**实现的方式**:

禁止不同的监视器互相依赖，digest循环只要运行一次就好了。

单向数据流好处不仅仅于此，更简单的数据流，而且性能提高了

3.增强angular1的脏值检测

我们说到digest循环有比较的步骤，比较操作的最佳算法是根据表达式返回值的类型进行比较。angular1有的是浅比较，有的是深比较，而且开发者并不能自己预先决定使用哪种比较。angular2团队把比较功能分离到了differ(差异比较器)中，那么开发者就可以通过2个基础类扩展自定义算法了，从而有了对脏值检测机制的完全控制。

●KeyValueDiffer。键值对型差异比较器。

●IterableDiffer。迭代型差异比较器。
---
title: '解读webpack的bundle.js'
date: 2019-09-16 22:44:39
tags: webpack
categories: webpack
copyright: true
---

可能就是好奇心略重了，读了一下webpack打包后的bundle.js的代码，复杂的模块可能读不懂，但简单的hello world模块我还是能看懂的。没什么目的，就是想通过几个简单的模块，一条简单的webpack命令，一个神奇的bundle.js代码来了解webpack是怎么把遵循commonJs规范的模块应用到浏览器端的。

几个简单的模块：
![img](https://rzaliyun.oss-cn-beijing.aliyuncs.com/blog/bundle_1.jpg?x-oss-process=style/demo)

一条简单的webpack命令：
![img](https://rzaliyun.oss-cn-beijing.aliyuncs.com/blog/bundle_2.jpg?x-oss-process=style/demo)

一个神奇的bundle.js：

无非就是一个立即执行的函数表达式嘛（IIFE），这是JavaScript中常见的独立作用域的方法。这里我将注释简化一下：

```
 1 (function(modules){
 2     //module缓存对象
 3     var installedModules = {};
 4     //require函数
 5     function __webpack_require__(moduleId){
 6         //检查module是否在cache中
 7         if(installedModules[moduleId]){
 8             return installedModules[moduleId].exports;
 9         }
10         //若不在cache中则新建module并放入cache中
11         var module = installedModules[moduleId] = {
12             exports: {},
13             id: moduleId,
14             loaded: false
15         };
16         //执行module函数
17         modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
18         //标记module已经加载
19         module.loaded = true;
20         //返回module的导出模块
21         return module.exports;
22     }
23 
24     //暴露modules对象（__webpack_modules__）
25     __webpack_require__.m = modules;
26     //暴露modules缓存
27     __webpack_require__.c = installedModules;
28     //设置webpack公共路径__webpack_public_path__
29     __webpack_require__.p = "";
30     //读取入口模块并且返回exports导出
31     return __webpack_require__(0);
32     
33 })([function(module, exports, __webpack_require__){ /*模块Id为0*/
34     var text = __webpack_require__(1);
35     console.log(text);
36 },function(module, exports){ /*模块Id为1*/
37     module.exports = 'Hello world';
38 }]);
```

这个IIFE接收一个数组作为参数modules，数组的每一项都是一个匿名函数代表一个模块（index.js和hello.js模块）。可是为什么构建命令中只出现了“index.js"呢，这是因为webpack通过静态分析index.js文件得到语法树，递归检测index.js所依赖的模块，以及依赖的依赖，入口模块和所有的依赖模块作为数组中的项，并合并到最终的代码中。

正是由于模块代码被包装成函数后，每个模块的运行时机变的可控，可以决定何时调用（通过 modules[moduleId].call ），而且也有了独立的作用域，定义变量，声明函数都不会污染全局作用域。模块函数的参数列表中除了commonJS规范所要求的 module 与 exports 外，还有 __webpack_require__ 函数，用来替换 require ，作用是声明对其他模块的依赖并获得该模块的 exports （ return installedModules[moduleId].exports 和 return module.exports; ）。而且不需要提供模块的相对路径或其他形式的ID，直接传入该模块在modules列表中索引即可，这样可以省掉模块标识符的解析过程（准确说，webpack是把require模块的解析过程提前到了构建期），从而可以获得更好的运行性能。



**程序解读：**
bundle.js通过 __webpack_require__(0); 启动整个程序，先检查模块ID = 0是否在缓存对象中，若该模块的缓存存在返回 module.exports 即模块所暴露出来的数据，若该模块的缓存不在则新创建module对象（该module对象作用是用来指向真实模块）并加入到缓存对象中，此时由于module对象和该模块的缓存对象 installedModules[moduleId] 的exports属性为没有数据，所以需要通过执行该模块函数来返回具体require其他模块的数据，传入的上下文对象是 module.exports 和 installedModules[moduleId].exports 所共同指向的一个对象。当程序执行到 var text = __webpack_require__(1); 时，又会执行 modules[1].call ，然后 module.exports = 'Hello world'; 将执行 __webpack_require__(1) 时创建的module1的exports赋值为Hello world，并返回，此时 __webpack_require__(1) 执行完毕，text为Hello world并打印， __webpack_require__(0) 执行完毕。这是一个递归的过程，如果还有更多依赖模块的话会更明显。

**总结一下，webpack主要做了两部分工作：**
1.分析得到所有必须模块并合并。
2.提供让这些模块有序，正常的执行环境。
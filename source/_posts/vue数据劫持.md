---
title: 'vue数据劫持'
date: 2019-09-16 23:48:01
tags: 数据劫持
categories: vue
copyright: true
---

# vue3.0怎么实现数据劫持的

## 什么是数据劫持

数据劫持通常我们利用`Object.defineProperty`劫持对象的访问器,在属性值发生变化时我们可以获取变化,从而进行进一步操作。

## 数据劫持的优势

1. 不用复杂的调用
2. 可以得知变化的数据

首先我们看下2.x用`Object.defineProperty`实现的简单的双向绑定

```js
const obj = {};
Object.defineProperty(obj, 'text', {
    get: function() {
        console.log('get val');
    },
    set: function(newVal) {
        console.log('set val:'+newVal)
        document.getElementById('input').value = newVal;
        document.getElementById('span').innerHTML = newVal;
    }
});
const oinput = document.getElementById('input');
oinput.addEventListener('keyup', function(e){
    obj.text = e.target.value;  
})
```

## Vue2.x的缺陷

### 属性值改为数组就无法监听数组变化

- 尤大神用了一些奇技淫巧,把无法监听数组的情况hack掉了

```js
const aryMethods = ['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse'];
const arrayAugmentations = [];
aryMethods.forEach((method)=> {
    // 这里是原生Array的原型方法
    let original = Array.prototype[method];
   // 将push, pop等封装好的方法定义在对象arrayAugmentations的属性上
   // 注意：是属性而非原型属性
    arrayAugmentations[method] = function () {
        console.log('我被改变啦!');
        // 调用对应的原生方法并返回结果
        return original.apply(this, arguments);
    };
});
let list = ['a', 'b', 'c'];
// 将我们要监听的数组的原型指针指向上面定义的空数组对象
// 别忘了这个空数组的属性上定义了我们封装好的push等方法
list.__proto__ = arrayAugmentations;
list.push('d');  // 我被改变啦！ 4
// 这里的list2没有被重新定义原型指针，所以就正常输出
let list2 = ['a', 'b', 'c'];
list2.push('d');  // 4
```

### 只能劫持对象的属性

假如需要对每个对象的每个属性进行遍历，如果属性值也是对象那么需要深度遍历,显然能劫持一个完整的对象是更好的选择。

## Vue3.x的特点

**带着以上缺点我们看下3.x版本怎么实现数据劫持**

> 3.x版本中把`Object.defineProperty`换成`Proxy`
>
> Proxy在ES6规范中被正式发布,它在目标对象之前架设一层“拦截”，外界对该对象的访问，都必须先通过这层拦截，因此提供了一种机制，可以对外界的访问进行过滤和改写,我们可以这样认为,Proxy是`Object.defineProperty`的全方位加强版
>
> 如果对`Proxy`属性还不是很了解的小伙伴 可以翻阅下Es6官方文档

### Proxy可以直接监听数组的变化

```js
const olist = document.getElementById('list');
const obtn = document.getElementById('btn');
// 渲染列表
const Render = {
  // 初始化
  init: function(arr) {
    const fragment = document.createDocumentFragment();
    for (let i = 0; i < arr.length; i++) {
      const li = document.createElement('li');
      li.textContent = arr[i];
      fragment.appendChild(li);
    }
    olist.appendChild(fragment);
  },
  // 我们只考虑了增加的情况,仅作为示例
  change: function(val) {
    const li = document.createElement('li');
    li.textContent = val;
    olist.appendChild(li);
  },
};
// 初始数组
const arr = [1, 2, 3, 4];
// 监听数组
const newArr = new Proxy(arr, {
  get: function(target, key, receiver) {
    console.log(key);
    return Reflect.get(target, key, receiver);
  },
  set: function(target, key, value, receiver) {
    console.log(target, key, value, receiver);
    if (key !== 'length') {
      Render.change(value);
    }
    return Reflect.set(target, key, value, receiver);
  },
});
// 初始化
window.onload = function() {
    Render.init(arr);
}
// push数字
obtn.addEventListener('click', function() {
  newArr.push(6);
});
```

### Proxy可以直接监听对象而非属性

```js
const input = document.getElementById('input');
const p = document.getElementById('p');
const obj = {};
const newObj = new Proxy(obj, {
  get: function(target, key, receiver) {
    console.log(`getting ${key}!`);
    return Reflect.get(target, key, receiver);
  },
  set: function(target, key, value, receiver) {
    console.log(target, key, value, receiver);
    if (key === 'text') {
      input.value = value;
      p.innerHTML = value;
    }
    return Reflect.set(target, key, value, receiver);
  },
});
input.addEventListener('keyup', function(e) {
  newObj.text = e.target.value;
});
```

## 总结

Proxy特点:

1. 可以劫持整个对象，并返回一个新对象
2. 有13种劫持操作
3. Proxy是es6提供的，兼容性不好,无法用polyfill磨平
4. 友情提示:目标对象内部的this关键字会指向 Proxy 代理
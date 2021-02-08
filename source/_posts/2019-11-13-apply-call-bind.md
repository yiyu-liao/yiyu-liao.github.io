---
layout: post
title:  "Apply/Call/Bind的一些用法"
date:   2019-11-13
catalog: true
tags: Javascript
---


## Apply/Call/Bind的用法

> 业务撸多了，最近突然想回顾下apply，call，bind的用法，佛系地记录一下哈

---

### 一、为什么需要Apply，Call，Bind？

Apply, Call, Bind属于function的prototype方法，允许我们改变函数的*this*指向，并通过对应的格式传参进行调用没错，这里你会问*this*是啥玩意？

* Everything in Javascript is an Object. * 

每个对象都分配有*this*，充当其对象的属性以及方法引用。

 ```js
 let obj = {
  name: "Alex",
  getName: function(friendName) { // 不要用（）=> {}, 箭头函数会让this指向getName函数对象，this.name会打印出undefind
    return `${friendName}, My Name is: ${this.name}`;
  }
}

obj.getName('Bob'); // "Bob, My Name is: Alex"
```

*this.name*是obj对象中name字段的引用，*this*就是整个obj的上下文（content）引用，大概这样理解吧。。。当我们去访问一个普通对象属性时候，通常用法*object.key*, 如上诉代码，调用obj.getName('Bob')时候，打印出obj对象中的name属性.

此时如果我想改变*this*指向呢（终于回到主题了），先说apply，后面会说三者的区别
 
```js
 let otherObj = { name: 'Lucas' }
 let obj = {
  name: "Alex",
  getName: function(friendName) {
    return `${friendName}, My Name is: ${this.name}`;
  }
}

obj.getName.apply(otherObj, 'Bob'); // "Bob, My Name is: Lucas"
```

apply传入了两个参数，第一个是参数为*this*指向的目标对象，剩下的参数为函数调用需要传入的参数


### 二、不同之处

* Apply, Call方法会改变目标函数的this指向，并立刻调用，Bind方法不会
* Apply, Call传参方式不同，Apply目标函数的参数以数组形式传入: fn.apply(targetContext,[ags]), Call则是以单个参数形式: n.call(targetContext, arg1, arg2)


### 三、一些编程技巧

通过Apply，Call, Bind方法，在对象中引入其他对象的一些特有属性（很重要）, 例如array类型可以引入Math的max方法求最大值，在string引入array的filter方法。

🌰借助Math.max在数组中找到最大值

```js
let a = [1, 2, 3, 4]
let max = Math.max.call(null, a)
console.log(max) // 4
```

🌰string类型通过call方法实现filter

```js
let letters = "abcd"
let new = Array.property.filter.call(letters, letter => {
    return letter !== a;
})
console.log(new) // ['b', 'c', 'd']
```

### Other

我们在编写vue组件，都会在lifehook（created/mounted/methods...)中编写定义好相关逻辑。组件
可以被多处地方复用，这时候vue源码内部会通过`bind(this)`方式保证各个组件具备独立的执行环境，避免造成上下文混乱。

---

完。

### 参考文章

[Call, Apply, Bind - The Basic Usages](https://dev.to/alexantoniades/call-apply-bind-the-basic-usages-5gpl)




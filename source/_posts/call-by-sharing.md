title: call by sharing
author: libaogang
tags:
  - javascript
  - java
  - 函数
categories:
  - 技术
thumbnail: /img/random/material-2.png
date: 2017-07-28 19:50:00
---
> 在了解到call by sharing这种函数参数传递机制之前，误以为javascript函数参数传递机制采用的是call by value （值传递）和call by reference（引用传递）。本文将分析下call by sharing这种参数传递机制。

## 数据类型及内存分配
javascript中的数据类型：
- 原始据类型primitive type ，比如Undefined,Null,Boolean,Number，String
- 引用类型 Object type ，比如Object,Array,Function,Date

不同数据类型的内存分配：
- 原始类型：存储在栈中的简单数据段。因为原始数据类型占据的内存空间是固定的，所以它们的值是直接存储在变量访问的位置即栈中，便于迅速查找变量的值。
- 引用类型：存储在堆中的对象。引用数据类型大小经常发生变化，如果把引用类型的值直接放在栈中会降低变量查找的速度。所以存放变量的栈空间的值是该对象存储在堆内存中的地址。因为内存地址的大小是固定的，所以把它存储在栈中对变量的查找性能无任何负面影响。

## 值传递（call by value）vs引用传递（call by reference）
按值传递(call by value)是最常用的求值策略：函数的形参是被调用时所传实参的副本。修改形参的值并不会影响实参。

按引用传递(call by reference)时，函数的形参接收实参的隐式引用，而不再是副本。这意味着函数形参的值如果被修改，实参也会被修改。同时两者指向相同的值。

按值传递由于每次都需要克隆副本，对一些复杂类型，性能较低。两种传值方式都有各自的问题。按引用传递会使函数调用的追踪更加困难，有时也会引起一些微妙的BUG。

以下为c的一个例子解释值传递和引用传递

```javascript
void Modify(int p, int * q){ 
    p = 27; // 按值传递 - p是实参a的副本, 只有p被修改
    *q = 27; // q是b的引用，q和b都被修改 
} 
int main()
{
    int a = 1;
    int b = 1;
    Modify(a, &b); // a 按值传递, b 按引用传递, // a 未变化, b 改变了
    return(0);
}
```  
## 探寻javascript函数参数传值方式
以下代码可以看出js中基本类型是按值传递的

```javascript
var a = 1;
function foo(x) {
    x = 2;
}
foo(a);
console.log(a); // 仍为1, 未受x = 2赋值所影响
```  
再看以下代码，obj的属性被修改了，说明obj和o指向同一个对象，那这是否就能说明js中的引用类型就是按引用传递的呢？

```javascript
var obj = {x : 1};
function foo(o) {
    o.x = 2;
}
foo(obj);
console.log(obj.x); // 2  
```  
以下代码，如果是按引用传递，obj的值应该被修改为libaogang才对，但是事实并非如此，所以js中的引用类型并不是按引用传递。

```javascript
var obj = {x : 1};
function foo(o) {
    o = "libaogang";
}
foo(obj);
console.log(obj.x); // 仍然是1, obj并未被修改为libaogang 
```  
## 共享传递（call by sharing）
准确的说，JS中的基本类型按值传递，对象类型按共享传递的(call by sharing，也叫按对象传递、按对象共享传递)。最早由Barbara Liskov. 在1974年的GLU语言中提出。值得注意的是，Python、Java、Ruby等多种语言都采用此种求值策略。

该求值策略表现为：形参为实参引用的副本，通过形参可以改变实参对象的属性，但是改变形参本身不会影响实参。
看以下代码

```javascript
var obj1 = {
    value:'1'
};
var obj2 = {
    value:'2'
};

function changeStuff(obj){
    obj.value = '3';
    obj = obj2;
}
console.log(obj1.value); //'3'     
```  
当调用changeStuff（obj），形参obj为实参obj1的副本。即形参obj中存放的是obj1对象在堆中的内存地址，所以通过形参obj可以改变obj1对象的属性。而当改变形参obj的值时，并不会影响实参obj1的值。所以当执行obj=obj2时，obj1.value仍为3。
## 结论
综上，js的基本类型是按值传递，引用类型是按共享传递。
而按共享传递本质就是对象在堆内存中的地址的按值传递，所以也可以认为js中所有函数的参数都按值传递的。
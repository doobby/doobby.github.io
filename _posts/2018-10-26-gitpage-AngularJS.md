---
title: 前端探索之angularJS
layout: post
tags: AngularJS,前端
author: rbw
---


### 前言

此学习总结完全是新手探索，但也不能直接从头开始因为没时间，所以如果看出差错请发我邮箱1522467044@qq.com

### 概念介绍


**WHAT**: AngularJS是一个JavaScript框架。它是一个以 JavaScript 编写的库。

**WHEN**: Angularjs是由google员工Miško Hevery等人从2009年着手开发，后被Google收购，版本1.0在2012年发布。

**WHY**: AngularJS是为了克服HTML在构建应用上的不足而设计的。

### 怎么码代码？？？？？

#### AngularJS指令
AngularJS是以ng作为前缀的html属性

ng-app：告诉AngularJS当前元素是根元素，所以AngularJS应用都必须有一个根元素。每个html只能有一个根元素，如果有多个只有第一个会生效

ng-init：初始化程序变量

示例：

```
<div ng-init="name='bowenren'">
<p my name is <span ng-bind="name"></span></p>`
</div>
```

#### AngularJS的表达式

AngularJS表达式写在双括号中{{expression}}

AngularJS表达式与ng-bind很像，把数据绑定到html上

AngularJS在表达式的位置输出数据，把表达式当作变量

AngularJS的表达式可以包含文字，运算符和变量


#### AngualarJS的指令
**干啥的？** --> 扩展html的属性。
**长啥样？** --> 前面都有"ng-"前缀
**都有哪些常用的** --> 例子如下

1. ng-app：声明AngularJS所管辖的区域，指定了应用程序的根元素，一般写在html标签上或者body里,原则上一个页面只能有一个。

```<body ng-app=""></body>
```

2. ng-model：把元素值绑定到应用程序的变量上。
```<div ng-app="" ng-init="firstName='John'">
     <p>在输入框中尝试输入：</p>
     <p>姓名：<input type="text" ng-model="firstName"></p>
     <p>你输入的为： {{firstName }}</p>
</div>
```
小理解：感觉就是ng-model会赋予某个元素（比如输入框<input>）一个变量，告诉你输入的值就是这个变量的值，之后也可以把这个变量用到其他地方。


3.ng-bind：把程序的值输出到html页面。（感觉和ng-model听起来差不多。。。待搞懂）

```<p ng-bind="5+5+'Angular'"></p>	
```

4.ng-repeat：重复一个html元素。




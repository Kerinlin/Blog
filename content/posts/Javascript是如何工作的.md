---
title: "Javascript是如何工作的"
date: 2021-12-04T19:48:45+08:00
draft: false
---

# Javascript是如何工作的

>原文: https://javascript.plainenglish.io/how-javascript-works-a-visual-guide-515199eef837

这篇博客通过动画的形式展现js的执行过程，对理解js是如何工作的很有帮助。本篇翻译会对原文翻译有所删减，剔除一些废话，直接翻译核心的讲解部分。另外水平有限，如果翻译有不对的地方，欢迎指正！

## 执行上下文

> Everythig in Javascript happens inside an Execution Context

Javascript中的一些都发生在执行上下文中。

我希望大家都能记住这句话，因为它非常重要！你可以假设**执行上下文**是一个大的容器，当浏览器想要运行js代码时会调用它。在这个容器中有两个组件：

1. 内存组件
2. 代码组件

#### 内存组件

内存组件也叫变量环境，在内存组件中，变量和函数存储成 **键值对** 的形式

#### 代码组件

在代码组件中，代码会一行一行的执行，代码组件还有一个别名叫做 **执行线程**

![](https://z3.ax1x.com/2021/11/20/IO85iq.png)



javascript是同步的单线程语言。因为它一次只能以特定的顺序执行一个命令。

## 代码执行

```javascript
var a = 2;
var b = 4;

var sum = a + b;

console.log(sum);

```

在这段代码中，初始化了两个变量a,b,然后将a和b相加的值赋值给了变量sum,我们来看看这段代码是怎么执行的。

![](https://z3.ax1x.com/2021/11/20/IOG9SK.gif)

浏览器创建了一个全局执行上下文，全局上下文中包含了刚刚上文说到的内存组件还有代码组件。浏览器会分两个阶段运行这段代码。

1. 内存创建阶段
2. 代码执行阶段

### 内存创建阶段

在内存创建阶段，js会扫描所有的代码，并为变量和函数分配内存。对于变量来说，在变量创建阶段，它们会被存储成undefined。至于函数，js会保留整个函数代码，在后面的例子中会讲到。

![](https://z3.ax1x.com/2021/11/20/IOGuSf.gif)



### 代码执行阶段

在代码执行阶段，js会逐行遍历所有的代码。在内存中，直到a被分配为2之前，a的值一直都为undefined。b也是同样的。然后js会将a和b相加的值6，存入内存中。然后打印sum值，最后销毁全局执行上下文。



## 函数在执行上下文中是怎么被调用的

js中的函数与其他编程语言的函数工作方式是不同的。举个栗子🌰

```javascript
var n = 2;

function square(num) {
 var ans = num * num;
 return ans;
}

var square2 = square(n);
var square4 = square(4);
```

上面的函数square接收一个参数num并返回num的平方。

在内存创建阶段，js创建了一个全局执行上下文，并且给变量和函数分配内存。函数将会被整个存储在内存中。

![](https://z3.ax1x.com/2021/11/20/IOGW6O.gif)



在代码执行阶段，js分配2值给了变量a,然后遇到函数，此时函数已经分配了内存，会直接跳到第6行，当运行函数square时，js又会在全局执行上下文下创建一个新的执行上下文。

![](https://z3.ax1x.com/2021/11/20/IOJ9Nn.gif)

在新的执行上下文中又会经历上文提到的两个阶段。在新执行上下文的内存创建阶段，给num,ans分配内存。

![](https://z3.ax1x.com/2021/11/20/IOJZB4.gif)

分配完成后，开始代码执行阶段，将n此时的值2分配给num,然后计算平方后将值分配给ans,然后返回值，再分配给square2。函数完成返回后会立马销毁它的执行上下文。

![](https://z3.ax1x.com/2021/11/20/IOJGuD.gif)

然后同样的方式处理square4。

![](https://z3.ax1x.com/2021/11/20/IOJa4I.gif)

所有的代码执行完毕后，销毁全局执行上下文。



## 调用栈(call stack)

![](https://z3.ax1x.com/2021/11/20/IOJ5vT.png)

在js中调用函数时，js会创建一个执行上下文，当函数存在嵌套的情况是，执行上下文将变得非常复杂。js会通过**调用栈**管理执行上下文的创建与删除。

### 栈

栈是一个有序项目集合，只能在一端进行插入和删除操作。类似于堆书。

```javascript
function a() {
    function insideA() {
        return true;
    }
    insideA();
}
a();
```

上面代码中，创建了一个函数a，a函数里面又调用了一个返回true的函数insadeA。下面来分析下这段代码的运行过程

![](https://z3.ax1x.com/2021/11/20/IOYSKO.gif)

1. js创建一个全局执行上下文，然后分两阶段，给函数a分配内存，然后执行代码调用函数a。
2. 调用函数后，又创建一个新的执行上下文(second)，这个新的执行上下文，放置在全局执行上下文上面。
3. 执行second执行上下文的代码，是调用函数insideA
4. 然后创建新的执行上下文(third),放置在second上，然后分配内存，执行代码返回true;
5. 开始依次销毁third,second,全局global执行上下文。



## 最后

最近准备每周翻译一篇文章，文章已放入[github](https://github.com/Kerinlin/translate-article),如果有愿意加入翻译的同学，可与我联系。

# 使用内联缓存技术动态优化javascript代码（动态优化or优化动态）

第一句 空

这是一个关于该优化技术的概述，这样的优化技术我在JSIL中使用了一段时间。为了在代码发生未预料到的改变时仍能保持快速的运行速度，JSIL在javascript运行时创建并更新多态内联缓存。

## 什么是内联缓存

你可能已经很熟悉内联缓存的概念，这一概念也被称作“PIC”，即多态内联缓存的简写。这一术语通常用于对JIT编译器（即时编译器）的讨论中，这样的即时编译器类似于在SpiderMonkey和V8 javascript引擎运行中使用的编译器。

如果你已经知道内联缓存是什么，请跳过这一节去下一节.下面我们来为初学者介绍一下，多态内联缓存进行一系列判断并且为已知、特定的场景生成特定的代码。在生成后的特定代码中，不包含有之前进行的一系列判断。

单从概念上来说，这一定义适用于更简单的场景，下面我们举一个简单的例子：
```
function divide (lhs, rhs) {
  return lhs / rhs;
}

function divideSomeNumbers (lhsArray, divisor, resultArray) {
  if (lhsArray.length !== resultArray.length)
    throw new Error("Arrays must be the same size");

  for (var i = 0, l = lhsArray.length; i < l; i++) {
    resultArray[i] = divide(lhsArray[i], divisor);
  }
}
```
这个例子很简单，有很多办法可以优化它。我们来做一个有趣的优化来说明在这个场景中，在这样使用比特位移做除法的运算中，内联缓存会产生什么样的代码。

当一个整数除以2的幂时，我们可以用二进制移位来得到与普通除法相同的结果。二进制移位操作在实现上相当简单，所以当等号右侧的除数时2的幂时，编译器就可能将这种除法操作转换为二进制移位。而如果我们要自己实现这种二进制移位的操作，那么我们就需要在代码中加入一部分查找来确定需要移动的比特位数：
```
var dividers = {
  2: function divideBy2 (lhs, unused) {
    return lhs >> 1
  },
  4: function divideBy4 (lhs, unused) {
    return lhs >> 2
  },
  undefined: function divideByNumber (lhs, rhs) {
    return lhs / rhs
  }
}

function divideSomeNumbers (lhsArray, divisor, resultArray) {
  if (lhsArray.length !== resultArray.length)
    throw new Error("Arrays must be the same size");

  var divider = dividers[divisor];

  for (var i = 0, l = lhsArray.length; i < l; i++) {
    resultArray[i] = divider(lhsArray[i], divisor);
  }
}
```
看上去不错对吧？但是我们把一个确定的函数调用替换成了一个在运行时才能确定的函数调用，也就是说具体调用哪个函数取决于我们发起调用的函数中的参数。这样的操作有可能会让代码运行速度减慢，除非JIT（即时编译器）足够智能地将每次对于divider方法的调用都对应到某个具体函数的调用。这种不确定性可能会影响代码内联，仅而影响其他重要的优化策略的实施。

让我们看看编译器使用内联缓存策略生成的代码的简易版本吧~
```
function divideByNumber (lhs, rhs) {
  return lhs / rhs;
}
function divideBy2 (lhs) {
  return lhs >> 1;
}
function divideBy4 (lhs) {
  return lhs >> 2;
}

function divideSomeNumbers (lhsArray, divisor, resultArray) {
  if (lhsArray.length !== resultArray.length)
    throw new Error("Arrays must be the same size");

  // Inline cache
  if (divisor === 2) {
    return divideSomeNumbersBy2(lhsArray, resultArray);
  } else if (divisor === 4)
    return divideSomeNumbersBy4(lhsArray, resultArray);
  } else {
  // 缓存失效!JIT可能会记录此处的缓存失效，并且考虑更新缓存。
  // 最终会发现：是否大多数经过此处的代码都没有命中缓存，或者是否缓存命中太多次？
  // 这时，出于对性能的考虑，内联缓存可能完全被删除掉。
    return divideSomeNumbersByUnknown(lhsArray, divisor, resultArray);
  }
}

function divideSomeNumbersBy2 (lhsArray, resultArray) {
  for (var i = 0, l = lhsArray.length; i < l; i++) {
    resultArray[i] = divideBy2(lhsArray[i]);
  }
}

function divideSomeNumbersBy4 (lhsArray, resultArray) {
  for (var i = 0, l = lhsArray.length; i < l; i++) {
    resultArray[i] = divideBy4(lhsArray[i]);
  }
}

function divideSomeNumbersByUnknown (lhsArray, divisor, resultArray) {
  for (var i = 0, l = lhsArray.length; i < l; i++) {
    resultArray[i] = divideByNumber(lhsArray[i], divisor);
  }
}

```
此时动态调用divider的代码已经被改成这样：我们在函数顶部检查是否能调用任何一个已经存在的二进制移位方法，如果可以，我们就调用那个确定的方法。因为调用对象是一个已经存在的方法，所以这样的调用并不是动态的。正因如此，下面的工作对编译器来说就很简单。编译器从进行检查的部分开始，内联所有代码，最终得到了一个函数，这个函数对于除数是2的幂的情况做了非常好的优化操作。

内联操作使得所有强大的优化策略成为可能，所以如果你想设置优化器，那么这样的内联缓存是必须的。而设置了优化器之后，编译器又能够做更多的优化操作，代码也能跑的更快。

## 为什么要在javascript中使用内联缓存？

JSIL是一个将使用类似c#的静态类型语言编写的.NET应用程序转换为javascript的编译器。在运行时，它实现了.NET的大多数语法，包括用户定义的值类型和方法重载。对于简单的应用来说，这种转换很简单，因为第JSIL最开始是由c#的反编译器发展而来的，之后逐渐变成了一个将原有代码转换为javascript语法的编译器。但是当我们想要转换更复杂的应用程序，并且让转换之后的应用程序运行更快的时候，事情就变得复杂起来。在.NET中有些特定操作并不能轻易转换到javascript中，而且将一个.NET应用暴露给javascript用户而不是转换整个应用程序也是非常困难的。下面是c#中的一个重载方法：
```
float Divide (float lhs, float rhs) {
  return lhs / rhs;
}

int Divide (int lhs, int rhs) {
  return lhs / rhs;
}

void Test () {
  float a = Divide(1.5, 3);
  int b = Divide(5, 2);
}
```
在javascript中没有等价的操作来表示这种重载。对于编译器来说，它可能会给重载的方法赋不同的名字，然后把它们都写到输出代码中：
```
function float_Divide_float_float (lhs, rhs) {
  return +((+lhs) / (+rhs));
}

function int_Divide_int_int (lhs, rhs) {
  return ((lhs | 0) / (rhs | 0)) | 0;
}

function Test () {
  var a = float_Divide_float_float(1.5, +3);
  var b = int_Divide_int_int(5, 2);
}
```
这不是什么好的解决方法，但它至少解决了问题。但是这个方法还有一个缺点。

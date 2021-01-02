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

虽然能用，但这不是什么好的解决方法，因为这个方法允许了静态类型的语言，在运行时操作变量类型和方法，这很奇怪。

c#中有个概念叫做“泛型类型”。在定义一个抽象泛型类型给一个变量的时候，这个变量的真实类型可能有很多种情况，之后在代码中将泛型类型和某些真实类型结合，来得到一个可用的真实类型。举个栗子，泛型类型list<T>结合了类型参数int产生了真实类型list<int>，这个类型是一个包含很多int类型数据的列表。这一切都是静态运行的，不会有任何的运行时类型检查，数组也会高效运行。一切看起来都很棒~~
  
然而

c#中还有个概念叫做“反射”。反射就意味着你必须在运行时检查变量类型、检查方法和值，然后操作它们。在.NET中，你甚至可以在运行时创造一个新的真实泛型类型，然后创建一个它的实例。当你做这样操作的时候，.NET的JIT（即时编译器）通过为这个新的类型重新编译一段代码来满足你的请求，这样就可以使用这个新的类型了。

但是这个时候，我们就发现，混淆生成一系列javascript函数的名字的想法可能并不奏效了。走到这里，我们发现，存在一些类似上述描述到的情景，能写一个正确地处理所有情况的编译器并不容易。那么我们应该怎么办？

## 使用内联缓存。但为什么选择它？

现在，放弃之前的观点，我们发现有些操作必须在运行时才能决定，因为即使对于c#这样的静态类型语言而言，这些操作的确定也是在运行时阶段确定的。确定这样的观点之后，我们就得想想如何设计代码来实现这样的需求。

在此之前，我们依靠编译器完成一系列变量提升等优化，现在在javascript中，我们得自己高效地实现这一系列操作了。虽然这种情况并不常见，但是c#中的常用的特性-接口-有时候也需要这种神奇的操作。下面我们来写一个简单的c#接口：
```
public interface ValueProvider<out T> {
  T GetValue();
}

public class GenericProvider<T> : ValueProvider<T> {
  public T TValue;
  
  T ValueProvider<T>.GetValue () {
    return TValue;
  }
}
```

调用代码如下：
```
public static class Program {
  public static void Main () {
    var provider = new GenericProvider<string> {
      TValue = "string"
    };
    
    ValueProvider<string> stringProvider = provider;
    Console.WriteLine(stringProvider.GetValue()); // prints "string"
  }
}
```

但下面的调用代码同样也能得到相同的结果：
```
public static class Program {
  public static void Main () {
    var provider = new GenericProvider<string> {
      TValue = "string"
    };
    
    ValueProvider<object> objectProvider = provider;
    Console.WriteLine(objectProvider.GetValue()); // prints "string"
  }
}
```

使第二种调用代码同样可以工作的关键特性叫做“变异”，关于变异更多的信息请参考[MSDN文档](https://docs.microsoft.com/en-us/dotnet/standard/generics/covariance-and-contravariance?redirectedfrom=MSDN)。

在编写代码的时候，编译器可以将原有代码中的类型定义替换为更具体或者更不具体的定义。在上面的例子中，由于string类型比object类型更为具体，所以接口

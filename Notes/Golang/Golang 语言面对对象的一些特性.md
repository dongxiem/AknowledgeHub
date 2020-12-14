# 前言

GO语言到底是不是面向对象的语言？如果你习惯了从某种特定编程语言的视角来思考问题，那么在这个话题上，你可能会由于自己所惯于使用的语言不同而得出截然相反的答案。

比如说，如果你之前习惯了使用C语言，那么很显然GO语言拥有太多面向对象的特性；但是如果你之前习惯了使用Java，那么GO语言看起来就不那么”面向对象”了。

因此关于这个话题，你首先要做的是摒弃其他语言带给你的先入为主的观念并学会用GO语言的思维模式来思考。

然后我就可以给出我的答案了：<u>**YES，GO语言是一门面向对象的语言，而且是以一种很清爽的方式实现了面向对象。**</u>

而在GO语言官方文档的FAQ中也表述过下述内容：

> **Is Go an object-oriented language?**
>
> **Yes and no**. Although Go has types and methods and allows an object-oriented（面对对象） style of programming, there is no type hierarchy. The concept of “interface” in Go provides a different approach that we believe is easy to use and in some ways more general. There are also ways to **embed types in other types to provide something analogous**—but not identical—to subclassing（子类化）. Moreover, methods in Go are more general than in C++ or Java: they can be defined for any sort of data, even built-in types such as plain, “unboxed” integers. They are not restricted to structs (classes).
>
> Also, the lack of a type hierarchy makes “objects” in Go feel much more lightweight than in languages such as C++ or Java.

简单翻译如下：

> 既是也不是。尽管Go拥有类型和方法，也允许面向对象风格的编程，但它没有类型层级。 在Go中“接口”的概念提供了不同的方法，我们相信它易于使用且在某些方面更通用。 也有一些在其它类型中嵌入类型的方法，来提供类似（而非完全相同）的东西进行子类化。 此外，Go中的方法比C++或Java中的更通用：它们可被定义为任何种类的数据。 甚至是像普通的“未装箱”整数这样的内建类型。它们并不受结构（类）的限制。
>
> 此外，类型层级的缺失也使Go中的“对象”感觉起来比C++或Java的更轻量级。



下面是对这几个面向对象的特性的认识

# 面对对象的三大特性



## **封装**

我认为GO语言中最好的一个特性就是，直接明了地通过字段名，方法名以及函数名的首字母大写来保证它们的访问可见性为public。

其他的那些以小写字母开头的字段等则默认为包(package)内私有(private)并且无法被导出至包外。

这个特性使得开发者们一眼就可以看出来哪些字段/方法/属性是public的，哪些是private的。

另外，由于**GO语言中没有继承这一概念**，所以GO语言中的访问修饰也就根本没有protected的概念.



## **多态**

定义：**多态是指代码可以根据类型的具体实现采取不同行为的能力**。

因为任何用户定义的类型都可以实现任何接口，**所以通过不同实体类型对接口值方法的调用就是多态**。

虽然Go里面也没有implements这样的关键字，但是在**Go里面可以使用interface来实现多态效果**，而且Go里面的接口相当灵活。

示例代码如下：

![image-20201214184222006](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20201214184222.png)



## **继承**

**GO语言中没有继承**，这一点在GO语言官方文档的FAQ也有明确说明：

> Object-oriented programming, at least in the best-known languages, involves too much discussion of the relationships between types, relationships that often could be derived automatically. **Go takes a different approach**.
>
> Rather than requiring the programmer to declare ahead of time that two types are related, **in Go a type automatically satisfies any interface that specifies a subset of its methods**. Besides reducing the bookkeeping, this approach has real advantages. Types can satisfy many interfaces at once, without the complexities of traditional multiple inheritance. Interfaces can be very lightweight—an interface with one or even zero methods can express a useful concept. Interfaces can be added after the fact if a new idea comes along or for testing—without annotating the original types. Because there are no explicit relationships between types and interfaces, there is no type hierarchy to manage or discuss.
>
> It's possible to use these ideas to construct something analogous to type-safe Unix pipes. For instance, see how fmt.Fprintf enables formatted printing to any output, not just a file, or how the bufio package can be completely separate from file I/O, or how the image packages generate compressed image files. All these ideas stem from a single interface (io.Writer) representing a single method (Write). And that's only scratching the surface. Go's interfaces have a profound influence on how programs are structured.
>
> It takes some getting used to but this implicit style of type dependency is one of the most productive things about Go.

简单翻译如下：

> 面向对象编程，至少在最著名的语言中，涉及了太多在类型之间关系的讨论， 关系总是可以自动地推断出来。Go则使用了一种不同的方法。
>
> 不像需要程序员提前声明两个类型的关联，在Go中类型会自动满足任何接口，以此实现其方法的子集。除了减少记账式编程外，这种方法拥有真正的优势。类型可立刻满足一些接口，而没有传统多重继承的复杂性。接口可以非常轻量——带一个甚至零个方法的接口能够表达一个有用的概念。若出现了新的想法，或为了测试目的，接口其实可以在以后添加——而无需注释掉原来的类型。 由于在类型和接口之间没有明确的关系，也就无需管理或讨论类型层级。
>
> 用这些思想来构造一些类似于类型安全的Unix管道是可能的。例如，看看 fmt.Fprintf 如何能将格式化打印到任何输出而不只是文件，或 bufio 包如何能从文件I/O中完全分离，或 image 包如何生成已压缩的图像文件。所有这些想法都来源于一个单一的接口 （io.Writer），都由一个单一的方法来表现（Write）。而这只是表面文章。Go的接口在如何组织程序方面有着深刻的影响。
>
> 它需要一段时间来适应，但这种隐式的类型依赖是Go中最具生产力的东西之一。

所以我们可以使用组合来替代继承：

程序设计理念中最广为认知的准则之一即是 用组合代替继承。这一准则在四人帮的那本极富盛名的《设计模式》也屡有提及。而在GO语言中，这一准则被发挥得淋漓尽致。

当我们在定义一个结构体时，我们**可以追加类型为另一个结构体的匿名字段**。这样一来，我们定义的这个结构体也就同时拥有了另一个结构体的所有字段以及方法。这种技法被称之为**Struct Embedding**。

示例代码如下：

![image-20201214184524690](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20201214184524.png)








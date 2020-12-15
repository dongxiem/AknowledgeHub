

想要了解这个 Receiver 是什么，可以看看在 [A Tour of Go ](https://tour.golang.org/methods/4) 里面是怎么说的：

> Pointer receivers
>
> You can declare methods with pointer receivers.
>
> This means the receiver type has the literal syntax *T for some type T. (Also, T cannot itself be a pointer such as *int.)
>
> For example, the Scale method here is defined on *Vertex.
>
> Methods with pointer receivers can modify the value to which the receiver points (as Scale does here). Since methods often need to modify their receiver, pointer receivers are more common than value receivers.
>
> Try removing the * from the declaration of the Scale function on line 16 and observe how the program's behavior changes.
>
> With a value receiver, the Scale method operates on a copy of the original Vertex value. (This is the same behavior as for any other function argument.) The Scale method must have a pointer receiver to change the Vertex value declared in the main function.

简单翻译如下：

> 可以声明一个带有指针接收器的方法。
>
> 这意味着接收器类型对于某些类型T具有字面语法* T（此外，T本身不能是* int之类的指针。）
>
> 例如，下面代码中的的Scale方法的接收器是 * Vertex。
>
> 带有指针接收器的方法可以修改接收器指向的值（就像下面的Scale所做的那样），由于一些方法通常需要修改其接收者的内容，因此指针接收器比值接收器更为普遍。
>
> 尝试从第16行的Scale函数的声明中删除 *，并观察程序的行为如何变化。
>
> 对于值接收器，Scale 方法操作的是从原 Vertex 值的一个复制副本进行操作（这与其他任何函数参数的行为相同。）Scale方法必须具有指针接收器才能更改在主函数中声明的Vertex值。

这个是 Golang 采用的一种方法，**当调用一个函数时，会对其每一个参数值进行拷贝**，如果**一个函数需要更新一个变量，或者函数的其中一个参数实在太大我们希望能够避免进行这种默认的拷贝，这种情况下我们就需要用到指针了**。对应到我们这里用来更新接收器的对象的方法，当这个接受者变量本身比较大时，我们就可以用其指针而不是对象来声明方法。

接收器有两种类型，一种是**指针还是非指针类型时**。在**声明一个method的receiver该是指针还是非指针类型时**，你需要考虑两方面的因素：

1. 如果这个对象本身是不是特别大，如果**声明为非指针变量时，调用会产生一次拷贝**；
2. 如果你用指针类型作为 receiver，那么你一定要注意，**这种指针类型指向的始终是一块内存地址**，就算你对其进行了拷贝。

可能你会看到很多指针的用法，一下子要取地址，一下子要解引用，一头雾水，简直不知道到底该用何种方法，不明白如果 method 里面的 receiver 是指针类型的话，需要通过指针类型/非指针类型就行调用，又或者 method 里面的 receiver 是非指针类型，能不能通过指针类型进行调用，Golang 这里做了一些手法：

- **不管你的 method 的 receiver 是指针类型还是非指针类型，都是可以通过指针/非指针类型进行调用的，编译器会帮你做类型转换**。

示例代码如下：

```go
package main

import (
    "fmt"
    "math"
)

type Vertex struct {
    X, Y float64
}

// 声明一个值接收器
func (v Vertex) Abs() float64 {
    return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

// 声明一个 指针接收器
func (v *Vertex) Scale(f float64) {
    v.X = v.X * f
    v.Y = v.Y * f
}

func main() {
    v := Vertex{3, 4}
    v.Scale(10)
    fmt.Println(v.Abs())
}
```

所以说，这样的一个约定可以很方便的满足我们的代码的编写，**比如在每一个合法的方法调用表达式中，也就是下面三种情况里的任意一种情况都是可以的**：

1. 两者都是类型`T`或者都是类型`*T`，也即是说接收器的实际参数和其形式参数此时是相同：

   - ```go
     Point{1, 2}.Distance(q) // 接收器的实际参数和其形式参数都为：Point
     pptr.ScaleBy(2)         // 接收器的实际参数和其形式参数都为：*Point
     ```

2. 接收器实参是类型`T`，但接收器形参是类型`*T`，这种情况下编译器会隐式地为我们取变量的地址：

   - ```go
     p.ScaleBy(2) // implicit (&p)，此时接收器实参是类型 p，但是但接收器形参是类型 *p
     ```

3. 接收器实参是类型*T，形参是类型T。编译器会隐式地为我们解引用，取到指针指向的实际变量：

   - ```go
     pptr.Distance(q) // implicit (*pptr)，此时接收器实参是类型 *p，但是但接收器形参是类型 p
     ```



反过来试想一下，假如没有 `Golang` 提供的这个自动转换的便利写法，而我们还是需要去修改一个对象的参数该怎么办呢？

还是有以下的几种写法，比如官方示例给的代码中 `*Vertex` 指针接收器，那么我们该如何去调用？有以下几种方法：

1. 提供一个`Point`类型的指针：

   - ```go
     v := &Vertex{1, 2}
     v.ScaleBy(10)
     fmt.Println(*v) //"{10, 20}"
     ```

2. 通过一个中间指针取地址进行修改：

   - ```go
     v := Point{1, 2}
     vptr := &v
     vptr.ScaleBy(10)
     fmt.Println(v) //"{10, 20}"
     ```

3.  直接通过取地址进行指针操作：

   - ```go
     v := Point{1, 2}
     (&v).ScaleBy(10)
     fmt.Println(v) //"{10, 20}"
     ```

看起来就显得有些复杂，写着写着就混乱了，所以我觉得 `Golang` 提供的这种机制简直太可以了，省略了像 `C/C++` 那些指针的繁琐，大大的节省了时间。



一些注意事项：

1. 在声明方法时，如果一个类型名本身是一个指针的话，是不允许其出现在接收器中的，比如下面这个程序：

   - ```go
     type P *int // p 本身是一个指针
     func (P) f() { /* ... */ } // compile error: invalid receiver type，编译错误，非法接收类型
     ```

2. Nil也是一个合法的接收器类型。

   - 就像一些函数允许nil指针作为参数一样，方法理论上也可以用nil指针作为其接收器，尤其当nil对于对象来说是合法的零值时，比如map或者slice。

3. 如果命名类型`T`的所有方法都是用T类型自己来做接收器（而不是`*T`），那么拷贝这种类型的实例就是安全的；但是如果一个方法使用指针作为接收器，你需要避免对其进行拷贝，因为这样可能会破坏掉该类型内部的不变性。

   






























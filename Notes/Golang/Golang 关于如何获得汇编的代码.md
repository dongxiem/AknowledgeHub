

在阅读源码期间，有时候我需要查看汇编相关代码，然后这里总结了以下几种获得汇编相关代码的方法。

代码很简单，我只是使用字面量初始化新的切片的方法去创建一个新的切片，然后将这个切片的内容打印出来，代码如下：

```go
package main

import "fmt"

func main() {
	// 创建 int 切片
	s1 := []int{1, 2, 3}

	fmt.Println(s1, len(s1), cap(s1))
}
```



## 方法一: `go tool compile`

我们去到该项目的目录下，使用`go tool compile -N -l -S main.go`生成汇编代码，截图如下：

![image-20201216231954507](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20201216231954.png)

便可以得到该源文件的汇编代码了。

## 方法二: `go tool objdump`

首先先编译程序: `go tool compile -N -l main.go`,

使用`go tool objdump main.o`反汇编出代码 ：

截图如下：

![image-20201216232136473](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20201216232136.png)

中间红框部分，则为  `s1 := []int{1, 2, 3}` 的具体汇编代码部分了，行号什么的也都显示出来，非常方便进行定位。

又或者其实也并不想看整个流程的内容，只是想查看特定内容的反汇编代码，则可以使用`go tool objdump -s main main.o`反汇编特定的函数，图示如下：

![image-20201216232447602](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20201216232447.png)



## 方法三: `go build -gcflags -S`

使用`go build -gcflags -S main.go`也可以得到汇编代码，截图如下：

![image-20201216232641803](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20201216232641.png)



综上，我觉得三种方法都挺好用的，但是说如果需要去获得指定函数的汇编代码的话，选择第二种方法挺好的！










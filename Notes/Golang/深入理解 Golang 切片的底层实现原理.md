



# 1.切片数据结构



`src/cmd/compile/internal/types/type.go`

```go
// SliceHeader 表示一个运行时的 slice。
// 它不能保证安全性和可移植性，它的实现在以后的版本中可能会进行改变。
// 而且，Data 字段也不能够充分保证它所指向的数据不会被当成垃圾进行回收，
// 所以程序需要要维护一个独立的、类型正确的指针指向底层数据。
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```



# 2.切片初始化



`src/cmd/compile/internal/types/type.go`



```go
// NewSlice returns the slice Type with element type elem.
func NewSlice(elem *Type) *Type {
	if t := elem.Cache.slice; t != nil {
		if t.Elem() != elem {
			Fatalf("elem mismatch")
		}
		return t
	}

	t := New(TSLICE)
	t.Extra = Slice{Elem: elem}
	elem.Cache.slice = t
	return t
}
```





```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}
```



切片创建的几种方式：

1. 直接声明

   - ```go
     var slice []int
     ```

2. 通过下标的方式获得数组或者切片的一部分；

   - ```go
     slice := array[1:5] 或 slice := sourceSlice[1:5]
     ```

3. 使用字面量初始化新的切片；

   - ```go
     slice := []int{1,2,3,4,5}
     ```

4. 使用关键字 `make` 创建切片：

   - ```go
     slice := make([]int, 5, 10)
     ```

     

不过按照我个人使用的话，我用的比较多的是 `make` 的方式来进行切片的创建，那么上面这几种创建方式有什么不同呢？

## 直接声明

使用 `var slice []int` 这种方式进行创建 slice 是一个 nil slice，它的长度和容量都为 0。和 `nil` 比较的结果为 true。

需要注意的是 `nil slice` 和 `empty slice` 的区别，虽然 `empty slice` 的长度和容量都为 0，但是 `empty slice` 和 nil 的比较结果不为 `true`，比较结果为 `false`，这是因为所有的 empty slice 的数据指针都有一个指向地址：`0xc42003bda0`。

常常看到很多切片的声明方法，这里写清楚一些，创建切片为 `nil slice` 的有：

```go
var s1 []int
var s2 = *new([]int)	
```

创建切片为 `empty slice` 的方式有：

```GO
var s3 = []int{}
var s4 = make([]int, 0)
```

这里再了解一些 `nil slice` 和 `empty slice` ：

`nil slice`：有时候，程序可能需要声明一个值为 `nil` 的切片（称为 `nil slice`）。只要在声明时不做任何初始化，就会创建一个 `nil slice`，而在 `golang` 中，`nil slice` 是很常见的创建切片的方法。`nil slice` 可以用于很多标准库和内置函数。在需要描述一个不存在的 `slice` 的时候，`nil slice` 则会很好用。

`empty slice`：利用初始化，通过声明一个切片可以创建一个 `empty slice`，`empty slice` 在底层数组包含了 0 个元素，也没有分配任何存储空间，想要表示空集合时候 `empty slice` 非常有用，比如在数据库查询返回 0 个查询结果时。

但是不管是 `nil slice` 还是 `empty slice`，对其调用内置函数 `append`、`len`、`cap` 的效果都是一样的。

## 使用字面量初始化新的切片

具体的创建切片方式代码如下：

```go
// 创建字符串切片
// 长度和容量都为 3 个元素
slice := []string{"Red", "Blue", "Green"}

// 创建整型切片
// 长度和容量都为 3 个元素
slice := []int{10, 20, 30}
```

这种创建切片的方式会比较简单，而且非常容易和创建数组的方式进行切分：如果在 [] 运算符中指定一个定值，那么创建的就是数组而不是切片，只有不指定值的时候，才是创建切片，代码如下：

```go
// 创建有 3 个元素的整型数组
array := [3]int{10, 20, 30}

// 创建长度和容量都为 3 的整型切片
slice := []int{10, 20, 30}
```

还需要注意的是，可能会觉得这种方式只能要用多少就需要用多少个元素去填充这个字面量，其实还可以进行一种方式进行设置初始长度和容量，就是使用索引这一方法进行，代码如下：

```go
// 创建字符串切片
// 使用空字符初始化第 100 个元素
slice := []string{99: ""}
```

下面这个例子可以更加清楚，使用了索引号，直接赋值，而其他未注明的元素则默认 `0 值`。代码如下：

```go
package main

import "fmt"

func main() {
	s1 := []int{0, 1, 2, 3, 8: 100}
	fmt.Println(s1, len(s1), cap(s1))
}
```

运行结果：

```go
[0 1 2 3 0 0 0 0 100] 9 9
```

关于这个字面量的底层实现，我们可以看源码：`cmd/compile/internal/gc.slicelit`，如果我们使用字面量 `[]int{1, 2, 3}` 创建新的切片时，该函数在编译期间会将它展开成如下所示的代码片段：

```go
var vstat [3]int
vstat[0] = 1
vstat[1] = 2
vstat[2] = 3
var vauto *[3]int = new([3]int)
*vauto = vstat
slice := vauto[:]
```

而这些步骤在源码中也写得很清楚了，但是太冗长了，简单概括如下：

1. 根据切片中的元素数量对底层数组的大小进行推断并创建一个数组；
2. 然后将这些所有的东西都塞进一个静态数组中；
3. 创建一个同样指向 `[3]int` 类型的数组指针；
4. 将静态存储区的数组 `vstat` 赋值给 `vauto` 指针所在的地址；
5. 通过 `[:]` 操作获取一个底层使用 `vauto` 的切片；

而第 5 步中的 `[:]` 就是使用下标创建切片的方法，从这一点我们也能看出 `[:]` 操作是创建切片最底层的一种方法。

源码如下：

```go
func slicelit(ctxt initContext, n *Node, var_ *Node, init *Nodes) {
    // make an array type corresponding the number of elements we have
    // 根据切片中的元素数量对底层数组的大小进行推断并创建一个数组
    t := types.NewArray(n.Type.Elem(), n.Right.Int64Val())
    dowidth(t)

    if ctxt == inNonInitFunction {
        // put everything into static array
        // 将这些所有的东西都塞进一个静态数组中
        vstat := staticname(t)

        fixedlit(ctxt, initKindStatic, n, vstat, init)
        fixedlit(ctxt, initKindDynamic, n, vstat, init)

        // copy static to slice
        // 将静态数组的内容拷贝到一个切片当中
        var_ = typecheck(var_, ctxExpr|ctxAssign)
        var nam Node
        if !stataddr(&nam, var_) || nam.Class() != PEXTERN {
            Fatalf("slicelit: %v", var_)
        }
        slicesym(&nam, vstat, t.NumElem())
        return
    }

    var vstat *Node

    ...

    // make new auto *array (3 declare)
    // 创建一个同样指向 types 类型的数组指针；
    vauto := temp(types.NewPtr(t))

    // set auto to point at new temp or heap (3 assign)
    // 设置该指针指向一个新的 temp 或者 heap
    var a *Node
    if x := prealloc[n]; x != nil {
        // temp allocated during order.go for dddarg
        if !types.Identical(t, x.Type) {
            panic("dotdotdot base type does not match order's assigned type")
        }

        if vstat == nil {
            a = nod(OAS, x, nil)
            a = typecheck(a, ctxStmt)
            init.Append(a) // zero new temp
        } else {
            // Declare that we're about to initialize all of x.
            // (Which happens at the *vauto = vstat below.)
            init.Append(nod(OVARDEF, x, nil))
        }

        a = nod(OADDR, x, nil)
    } else if n.Esc == EscNone {
        ...
    } else {
        ...
    }

    a = nod(OAS, vauto, a)
    a = typecheck(a, ctxStmt)
    a = walkexpr(a, init)
    init.Append(a)

    if vstat != nil {
        // copy static to heap (4)
        // 将静态存储区的数组 vstat 赋值给 vauto 指针所在的地址；
        a = nod(ODEREF, vauto, nil)

        a = nod(OAS, a, vstat)
        a = typecheck(a, ctxStmt)
        a = walkexpr(a, init)
        init.Append(a)
    }

    // put dynamics into array (5)
    // 将一些动态的东西也放入数组当中
    var index int64
    for _, value := range n.List.Slice() {

        a := nod(OINDEX, vauto, nodintconst(index))
        a.SetBounded(true)
        index++
        ...

        // build list of vauto[c] = expr
        setlineno(value)
        a = nod(OAS, a, value)

        a = typecheck(a, ctxStmt)
        a = orderStmtInPlace(a, map[string][]*Node{})
        a = walkstmt(a)
        init.Append(a)
    }

    // make slice out of heap (6)
    // 最后获取底层的切片
    a = nod(OAS, var_, nod(OSLICE, vauto, nil))

    a = typecheck(a, ctxStmt)
    a = orderStmtInPlace(a, map[string][]*Node{})
    a = walkstmt(a)
    init.Append(a)
}
```









## 使用关键字 make 创建切片





# 3.切片元素访问





# 4.切片扩容与追加



4.1 扩容策略



4.2 是否覆盖





```go
// growslice handles slice growth during append.
// It is passed the slice element type, the old slice, and the desired new minimum capacity,
// and it returns a new slice with at least that capacity, with the old data
// copied into it.
// The new slice's length is set to the old slice's length,
// NOT to the new requested capacity.
// This is for codegen convenience. The old slice's length is used immediately
// to calculate where to write new values during an append.
// TODO: When the old backend is gone, reconsider this decision.
// The SSA backend might prefer the new length or to return only ptr/cap and save stack space.
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}

	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.cap < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	// The check of overflow in addition to capmem > maxAlloc is needed
	// to prevent an overflow which can be used to trigger a segfault
	// on 32bit architectures with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	if et.ptrdata == 0 {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem-et.size+et.ptrdata)
		}
	}
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```















# 5.切片拷贝







```go
// slicecopy is used to copy from a string or slice of pointerless elements into a slice.
func slicecopy(toPtr unsafe.Pointer, toLen int, fromPtr unsafe.Pointer, fromLen int, width uintptr) int {
	if fromLen == 0 || toLen == 0 {
		return 0
	}

	n := fromLen
	if toLen < n {
		n = toLen
	}

	if width == 0 {
		return n
	}

	size := uintptr(n) * width
	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(slicecopy)
		racereadrangepc(fromPtr, size, callerpc, pc)
		racewriterangepc(toPtr, size, callerpc, pc)
	}
	if msanenabled {
		msanread(fromPtr, size)
		msanwrite(toPtr, size)
	}

	if size == 1 { // common case worth about 2x to do here
		// TODO: is this still worth it with new memmove impl?
		*(*byte)(toPtr) = *(*byte)(fromPtr) // known to be a byte pointer
	} else {
		memmove(toPtr, fromPtr, size)
	}
	return n
}
```




























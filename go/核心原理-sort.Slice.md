本文章来源于：<https://github.com/Zeb-D/my-review> ，请star 强力支持，你的支持，就是我的动力。

[TOC]

------

### 背景

在搬砖的时候需要对一组数据排序，当时也看到pkg有对应的排序算法如`sort.Slice`，

平时业务上造轮子造多了，也就手撕了快速排序，

结果自己写的代码居然比自带库慢多了...



### 示例

```go
func partition(a [][]int, low, high int) int {
	rIdx := low + rand.Intn(high-low)
	if rIdx < high {
		a[low], a[rIdx] = a[rIdx], a[low]
	}
	pivot := a[low][1]
	j := low
	for i := low + 1; i <= high; i++ {
		if a[i][1] < pivot {
			j++
			a[i], a[j] = a[j], a[i]
		}
	}
	a[low], a[j] = a[j], a[low]
	return j
}

func quickSort(a [][]int, low, high int) {
	if low < high {
		idx := partition(a, low, high)
		quickSort(a, idx+1, high)
		quickSort(a, low, idx-1)
	}
}
```

测试[[无重叠区间](https://leetcode-cn.com/problems/non-overlapping-intervals/)]：

```go
func eraseOverlapIntervals(intervals [][]int) int {
	n := len(intervals)
	if n == 0 {
		return 0
	}
	quickSort(intervals, 0, n-1) //手撕的
	//sort.Slice(intervals, func(i, j int) bool { return intervals[i][1] < intervals[j][1] })
	end, cnt := intervals[0][1], 1
	for i := 1; i < n; i++ {
		if intervals[i][0] < end {
			continue
		}
		end = intervals[i][1]
		cnt++
	}
	return n - cnt
}
```

测试结果：

```
方法				耗时		内存
sort.slice 220 ms	15.9 MB
quickSort  360 ms	18.7 MB
```

比原生的差了不止一点点，慢了一半了；



### 原理

sort.Slice是golang提供的切片排序方法，

 其中使用到了反射（reflect）包：

```go
// Slice sorts the slice x given the provided less function.
// It panics if x is not a slice.
//
// The sort is not guaranteed to be stable: equal elements
// may be reversed from their original order.
// For a stable sort, use SliceStable.
//
// The less function must satisfy the same requirements as
// the Interface type's Less method.
func Slice(x interface{}, less func(i, j int) bool) {
	rv := reflectValueOf(x)
	swap := reflectSwapper(x)
	length := rv.Len()
	quickSort_func(lessSwap{less, swap}, 0, length, maxDepth(length))
}
```

使用了闭包

```go
var reflectValueOf = reflectlite.ValueOf
var reflectSwapper = reflectlite.Swapper

// ValueOf returns a new Value initialized to the concrete value
// stored in the interface i. ValueOf(nil) returns the zero Value.
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}

	// TODO: Maybe allow contents of a Value to live on the stack.
	// For now we make the contents always escape to the heap. It
	// makes life easier in a few places (see chanrecv/mapassign
	// comment below).
	escapes(i)

	return unpackEface(i)
}

// Swapper returns a function that swaps the elements in the provided
// slice.
//
// Swapper panics if the provided interface is not a slice.
func Swapper(slice interface{}) func(i, j int) {
	v := ValueOf(slice)
	if v.Kind() != Slice {
		panic(&ValueError{Method: "Swapper", Kind: v.Kind()})
	}
	// Fast path for slices of size 0 and 1. Nothing to swap.
	switch v.Len() {
	case 0:
		return func(i, j int) { panic("reflect: slice index out of range") }
	case 1:
		return func(i, j int) {
			if i != 0 || j != 0 {
				panic("reflect: slice index out of range")
			}
		}
	}

	typ := v.Type().Elem().(*rtype)
	size := typ.Size()
	hasPtr := typ.ptrdata != 0

	// Some common & small cases, without using memmove:
	if hasPtr {
		if size == ptrSize {
			ps := *(*[]unsafe.Pointer)(v.ptr)
			return func(i, j int) { ps[i], ps[j] = ps[j], ps[i] }
		}
		if typ.Kind() == String {
			ss := *(*[]string)(v.ptr)
			return func(i, j int) { ss[i], ss[j] = ss[j], ss[i] }
		}
	} else {
		switch size {
		case 8:
			is := *(*[]int64)(v.ptr)
			return func(i, j int) { is[i], is[j] = is[j], is[i] }
		case 4:
			is := *(*[]int32)(v.ptr)
			return func(i, j int) { is[i], is[j] = is[j], is[i] }
		case 2:
			is := *(*[]int16)(v.ptr)
			return func(i, j int) { is[i], is[j] = is[j], is[i] }
		case 1:
			is := *(*[]int8)(v.ptr)
			return func(i, j int) { is[i], is[j] = is[j], is[i] }
		}
	}

	s := (*unsafeheader.Slice)(v.ptr)
	tmp := unsafe_New(typ) // swap scratch space

	return func(i, j int) {
		if uint(i) >= uint(s.Len) || uint(j) >= uint(s.Len) {
			panic("reflect: slice index out of range")
		}
		val1 := arrayAt(s.Data, i, size, "i < s.Len")
		val2 := arrayAt(s.Data, j, size, "j < s.Len")
		typedmemmove(typ, tmp, val1)
		typedmemmove(typ, val1, val2)
		typedmemmove(typ, val2, tmp)
	}
}
```

可以参考何时使用panic

```go
length := rv.Len()

// Len returns v's length.
// It panics if v's Kind is not Array, Chan, Map, Slice, or String.
func (v Value) Len() int {
	k := v.kind()
	switch k {
	case Array:
		tt := (*arrayType)(unsafe.Pointer(v.typ))
		return int(tt.len)
	case Chan:
		return chanlen(v.pointer())
	case Map:
		return maplen(v.pointer())
	case Slice:
		// Slice is bigger than a word; assume flagIndir.
		return (*unsafeheader.Slice)(v.ptr).Len
	case String:
		// String is bigger than a word; assume flagIndir.
		return (*unsafeheader.String)(v.ptr).Len
	}
	panic(&ValueError{"reflect.Value.Len", v.kind()})
}
```

quickSort函数设计学习，同时使用了heapsort、insertionSort和单次希尔排序
说白了，不同的排序算法有不同的优势，局部优化从而达到全局优化；

```go

// Auto-generated variant of sort.go:quickSort
func quickSort_func(data lessSwap, a, b, maxDepth int) {
	for b-a > 12 {
		if maxDepth == 0 {
			heapSort_func(data, a, b)
			return
		}
		maxDepth--
		mlo, mhi := doPivot_func(data, a, b)
		if mlo-a < b-mhi {
			quickSort_func(data, a, mlo, maxDepth)
			a = mhi
		} else {
			quickSort_func(data, mhi, b, maxDepth)
			b = mlo
		}
	}
	if b-a > 1 {
		for i := a + 6; i < b; i++ {
			if data.Less(i, i-6) {
				data.Swap(i, i-6)
			}
		}
		insertionSort_func(data, a, b)
	}
}
```

合法性处理

```go
func Swapper(slice interface{}) func(i, j int) {
	v := ValueOf(slice)
	if v.Kind() != Slice {
		panic(&ValueError{Method: "Swapper", Kind: v.Kind()})
	}
	// Fast path for slices of size 0 and 1. Nothing to swap.
	switch v.Len() {
	case 0:
		return func(i, j int) { panic("reflect: slice index out of range") }
	case 1:
		return func(i, j int) {
			if i != 0 || j != 0 {
				panic("reflect: slice index out of range")
			}
		}
	}
```

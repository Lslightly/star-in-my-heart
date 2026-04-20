---
tags:
    - Go
---

# [goref](https://github.com/cloudwego/goref)

## 快速尝鲜

首先在根目录 `go install ./cmd/grf`

在 [`test/testdata/mockleak`](https://github.com/Lslightly/goref/tree/373c1be867ff466b27ca24d7c52b9b3211393719/test/testdata/mockleak) 目录下执行如下命令

```bash
go build -gcflags="-N -l" .
./mockleak # 等待一分钟(通过cmds.AttachSelf函数实现50000次时导出)，或者grf attach该进程
go tool pprof -http=127.0.0.1:8080 grf.profile.gz
```

得到下图：

不同类型的对象的大小会被展示。对象之间的引用关系也会被展示。

![attachments/Pasted image 20260116205231.png](attachments/Pasted%20image%2020260116205231.png)

top 如下：

![attachments/Pasted image 20260116205501.png](attachments/Pasted%20image%2020260116205501.png)

flame graph:

![attachments/Pasted image 20260116205601.png](attachments/Pasted%20image%2020260116205601.png)

source 视图展示的东西比较诡异。

![attachments/Pasted image 20260116205711.png](attachments/Pasted%20image%2020260116205711.png)

## goref 原理

goref 整体工作流程主要3步，如下图所示：
- [readHeap](#readHeap)
- [FindRef](#FindRef)，主要是根据dwarf.Type 扫描
- [FinalMark](#FinalMark)，仿照 GC 算法扫描

![[attachments/Pasted image 20260420210611.png]]

## readHeap

goref 首先会 `readHeap` 从 delve 中读取进程中的变量，恢复 go runtime 中的数据结构，方便 goref 做后续遍历。

`region` 的概念应该是区域的意思，包含的 `mem` 是 `delve/pkg/proc.MemoryReadWriter`

![[attachments/Pasted image 20260420211159.png]]


`(*HeapScope).readHeap` 会读取以下信息:
1. `mheap_`
2. `_PageSize`
3. `mSpanInUse`
4. `heapArenaBytes`
5. `_KindSpecialFinalizer`
6. `_KindSpecialCleanup`
7. `arenaL1Bits`, `arenaL2Bits`
8. `minSizeForMallocHeader`

也会通过 `(*HeapScope).readAllSpans` 读取所有的 span 信息。

`readHeap` 还会通过 `readArena` 从 `arena` 中读取 bitmap，但是对于 go1.22 添加了 allocation header 之后的实现，则会通过 `readTypePointers` 获取 allocation header 相关的内容。

在 `readTypePointers` 中，如果 `spanclass` 是 `noscan`，没有指针，则直接返回。
如果 span 中每个对象的 size 较小，那么 heapBits 就会放在 span 末尾。否则就通过 `largeType` 存放。

最后 `readHeap` 会通过 `readModuleData` 从 dwarf 中读取 module data.


### readAllSpans

[pkg/proc/heap.go#L221](https://github.com/cloudwego/goref/blob/4174f741fdea8f00b5229652c92fd56915411520/pkg/proc/heap.go#L221)
```go
		maskLen := CeilDivide2(spanSize, 8, 64)
```

这里除 8 再除 64 的原因是：
- 除 8 是因为指针是 8 字节
- 除 64 是因为使用 uint64 来保存 bitmap

### readTypePointers

`heapBitsInSpan` 在 span 结尾。`bitmapSize = spanSize / 8 / 8`, ptr size 8, 一个字节 8bit。

### readModuleData

`runtime.firstmoduledata` 的类型为 `moduledata`，注释如下：

[src/runtime/symtab.go#L389-394](http://222.195.92.204:1480/vm/golang/work/go1.24.2/-/blob/b9e570afd7a917147792d244e8609d98b421f03c/src/runtime/symtab.go#L389-394)
```go
// moduledata records information about the layout of the executable
// image. It is written by the linker. Any changes here must be
// matched changes to the code in cmd/link/internal/ld/symtab.go:symtab.
// moduledata is stored in statically allocated non-pointer memory;
// none of the pointers here are visible to the garbage collector.
type moduledata struct {
```


在读取完堆上的数据结构信息(`readHeap`)之后，就可以获取 ModuleData、全局变量、局部变量、Finalizer、Cleanup 等内容，实现遍历标记，构建引用图。

### getModuleData

通过 linkname 链接到 delve 中的 `pkg/proc.(*BinaryInfo).getModuleData`.

返回的类型 `([]proc.ModuleData, error)`。 delve 中的 ` proc.ModuleData `。

```go
type ModuleData struct {
	text, etext   uint64
	types, etypes uint64
	typemapVar    *Variable
}
```

## FindRef

![[attachments/Pasted image 20260420211241.png]]


### global var 处理

通过 `scope.PackageVariables` 获取全局变量，然后通过 `s.findRef(rv, nil)` 给 `rv` 添加信息。

```go
type ReferenceVariable struct {
	Addr     Address
	Name     string
	RealType godwarf.Type
	mem      proc.MemoryReadWriter

	// heap bits for this object
	hb *gcMaskBitIterator

	// node size
	size int64
	// node count
	count int64
}
```

### 局部变量处理

简单来说是从 `proc.GoroutineStacktrace` 中获取栈帧信息，然后从栈帧信息中获取局部变量。

### findRef

如果 `x.Name` 非空，有名字，那么就在 `pushHead` 到 `idx` 这个 pprof 列表中，pprof 列表的 max depth 有限制。然后在函数结尾添加一个注册函数 `s.record(idx, x.size, x.count)`
如果 `x.Name` 为空，表示新发现的堆对象，在函数结尾注册一个函数，如果还有下一个指针，则添加到 `finalMarks` 中。

在判断完 `x.Name` 之后，判断 `x` 的 `RealType godwarf.Type` 实际类型。

- `PtrType`

通过 `readPointer` 将 GCMask 置零。然后通过 `readUintRaw` 读取指针值 `ptrval`，然后对 `ptrval` 执行 `findObject`。

如果发现对象，就将这个对象 `y` 的 `y.size` 累计到 `x.size ` 中，并且将 `y.count` 也累加到 `x.count` 中。

- `ChanType`

`readPointer` 返回的 `ptrval` 应该是 `*hchan`，通过 `findObject` 得到 `hchan`
`zptrval` 是 `hchan.buf` 值
`chanLen` 是 `hchan.dataqsiz` 值

然后从 `hchan` 对应的 ring buffer 里找数组对象。

- `MapType`

`ptrval` 是 `*hmap`。获得得到的 `y` 是 `hmap` 结构体。

这个需要了解两种 map 的内部结构才能知道迭代方式是什么。

- `StringType`
读 `str` 数组和 `len` 长度。不过奇怪的是 string 对象也还是要 `findRef`。

- `SliceType`

读取 `base` 和 `cap_`，然后找数组对象。

- `InterfaceType`

根据 iface 中 `_type` 和 `Type` 类型决定是 `abi.Type` 还是 `*_type` 结构体。

如果是 `*_type` 结构体，就通过 `newVariable` 创建一个类型变量。

对于 indirect interface（即被赋值给 interface 的对象必须在堆上重新分配内存才能表示，不能用原始数据） 来说，`rtyp` 存储实际数据的指针类型。

itype 是整合得到的实际赋给 interface 包含或者指向的对象的类型。

例如 `*int` 类型，itype 就是 `int`，如果是 `struct {x, y int}`，就是 `struct`，如果是 `struct {p *int}`，就是 `int`。如果是 `**int`，那么就是 `*int` 类型。

然后通过 `ptrval` 这个指针和 `ityp` 访问对象。

- `StructType`

对每个域都做查找。

- `ArrayType`

对于 array 中的每个元素都做查找。对于索引在 10 以上的，统一用 `[10+]` 表示。

- `FuncType`

首先获取 closure 地址，closure 的第一个 field 是函数指针 `funcAddr`。通过 `s.bi.PCToFunc(funcAddr)` 获取 `proc.Function`。

通过 `pkg/proc.(*Function).extra` 获取 closure 的闭包结构体类型，这个结构体应该是包含了对其他对象的引用，继续对这个结构体做查找。

- `finalizePtrType`

通过 `findObject`，从 `x.Addr` 寻找对象。


## FinalMark

![[attachments/Pasted image 20260420211409.png]]


[pkg/proc/reference.go#L158](https://github.com/Lslightly/goref/blob/373c1be867ff466b27ca24d7c52b9b3211393719/pkg/proc/reference.go#L158)
```go
func (s *ObjRefScope) finalMark(idx *pprofIndex, hb *gcMaskBitIterator) {
```

先通过 `hb.nextPtr(true)` 利用 `gcMask` bits 找到下一个指针。

`ptr` 的值从 `cacheMemory` 函数返回的 `cmem` 中读取。

然后通过 `markObject` 函数标记地址。

### gcMaskBitIterator.nextPtr

通过 `gcMaskBitIterator.addr` 字段进行迭代，`startOffset` 是 `addr` 相比于 `maskBase` 的偏移，从 0 开始。`ptrIdx` 则为 `startOffset / 8 / 64`，代表 `mask []uint64` 的第几个 mask，`i = startOffset / 8 % 64` 表示 `mask[ptrIdx]` 中的第几位。

`j = bits.TrailingZeros64(b.mask[ptrIdx] >> i)`。
如果 i 是 0，则看最后一位是否为 0,。如果为 0，表示当前 i 所代表的不是指针，j 是下一个 gcmask 为 1 的地址和 i 所代表的地址之间的距离/8，这时候迭代器地址就会加 8。如果在 `b.mask[ptrIdx]` 中已经没有为 1 的了，就看下一个 `mask`。返回 0 就代表已经找不到下一个指针了。

### markObject

worklist 算法

伪代码如下：

```
for !stack.empty()
	addr = stack.pop()
	sp, base = findSpanAndBase(addr) # 找到这个地址所处的span和这个addr所处对象的基地址
	if sp == nil
		continue # span not found
	if !sp.mark(base)
		continue # 这个对象已经标记过了
	realBase = copyGCMask(sp, base) # 两个作用，纠正对象的基地址，更新sp.ptrMask。更新sp.ptrMask主要是靠readType实现的
	hb = GCBitsIterator{realBase, sp.elemEnd(base), sp.base, sp.ptrMask} # 创建GC迭代器，迭代器的mask就是sp.ptrMask
	for
		ptr = hb.nextPtr(true)
		if ptr == 0
			break
		stack.append(ptr)
```

### readType

`typeAddr` 主要是对象的 header 信息。包含 `typeSize`, `ptrBytes`, `gcDataAddr`

`addr` 后是对象的信息，记 `elem` 是对象的起始地址。通过 `gcDataAddr + (addr-elem)/64` 地址获取 mask。除 64 是因为除 8 (pointer size)，再除 8（byte, uint8）。获取到 mask 之后，更新 `ptrMask[offset/8/64]` 的 `offset/8%64` 位。

- 需要了解有这种 header 的情况下 header 内存了什么信息。


> [!Note]
> `FinalMark` 找到其他指针的示例。
> ```go
> type S struct {
> 	pa *int
> 	a int
> 	pb *S
> }
> ```
> 
> `s.pa = &s.a`，先遍历到 `s.pa`，然后发现是一个结构体，再扫描 `s.pb`，分析结构体其他域。




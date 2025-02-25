---
weight: 2302
title: "8.2 写屏障技术"
---

# 8.2 写屏障技术

屏障技术在本书指内存屏障（Memory Barrier）。
它保障了代码描述中对内存的操作顺序
**既不会在编译期被编译器进行调整，也不会在运行时被 CPU 的乱序执行所打乱**，
是一种语言与语言用户间的契约。

## 8.2.1 并行标记清理产生的问题

在没有用户态代码并发修改三色抽象的情况下，回收可以正常结束。但并发回收的根本问题在于，
用户态代码在回收过程中会并发的更新对象图，从而赋值器和回收器可能对对象图的结构产生不同的认知，
这时以一个固定的三色波面作为回收过程前进的边界则不再合理。

我们不妨考虑赋值器的写操作，假设某个灰色对象 A 指向白色对象 B，
而此时赋值器并发的将黑色对象 C 指向（ref3）了白色对象 B，
并将灰色对象 A 对白色对象 B 的引用移除（ref2），则在继续扫描的过程中，
白色对象 B 永远不会被标记为黑色对象了（回收器不会重新扫描黑色对象）。
进而产生被错误回收的对象 B，如图 1 所示。

<div class="img-center" style="margin: 0 30px 30px 50px; float: right; max-width: 30%">
<img src="../../../assets/gc-mutator.png"/>
<strong>图 1: 回收器正确性的破坏</strong>
</div>

## 8.2.2 弱三色不变性

垃圾回收器的正确性体现在：不应出现对象的丢失，也不应错误的回收还不需要回收的对象。
作为内存屏障的一种，写屏障（Write Barrier）是一个在并发垃圾回收器中才会出现的概念。

可以证明，当以下两个条件同时满足时会破坏垃圾回收器的正确性 [Wilson, 1992]：

- 条件 1: 赋值器修改对象图，导致某一黑色对象引用白色对象；
- 条件 2: 从灰色对象出发，到达白色对象的、未经访问过的路径被赋值器破坏。

只要能够避免其中任何一个条件，则不会出现对象丢失的情况，因为：

- 如果条件 1 被避免，则所有白色对象均被灰色对象引用，没有白色对象会被遗漏；
- 如果条件 2 被避免，即便白色对象的指针被写入到黑色对象中，但从灰色对象出发，总存在一条没有访问过的路径，从而找到到达白色对象的路径，白色对象最终不会被遗漏。

我们不妨将三色不变性所定义的波面根据这两个条件进行削弱：

- 当满足原有的三色不变性定义（或上面的两个条件都不满足时）的情况称为强三色不变性（strong tricolor invariant）
- 当赋值器令黑色对象引用白色对象时（满足条件 1 时）的情况称为弱三色不变性（weak tricolor invariant）

当赋值器进一步破坏灰色对象到达白色对象的路径时（进一步满足条件 2 时），即打破弱三色不变性，
也就破坏了回收器的正确性；或者说，在破坏强弱三色不变性时必须引入额外的辅助操作。
弱三色不变形的好处在于：**只要存在未访问的能够到达白色对象的路径，就可以将黑色对象指向白色对象。**

### 8.2.3 赋值器的颜色

如果我们考虑并发的用户态代码，回收器不允许同时停止所有赋值器，
就是涉及了存在的多个不同状态的赋值器。为了对概念加以明确，还需要换一个角度，
把回收器视为对象，把赋值器视为影响回收器这一对象的实际行为（即影响 GC 周期的长短），
从而引入赋值器的颜色：

- 黑色赋值器：已经由回收器扫描过，不会再次对其进行扫描。
- 灰色赋值器：尚未被回收器扫描过，或尽管已经扫描过但仍需要重新扫描。

赋值器的颜色对回收周期的结束产生影响：
如果某种并发回收器允许灰色赋值器的存在，则必须在回收结束之前重新扫描对象图。
如果重新扫描过程中发现了新的灰色或白色对象，回收器还需要对新发现的对象进行追踪，
但是在新追踪的过程中，赋值器仍然可能在其根中插入新的非黑色的引用，如此往复，
直到重新扫描过程中没有发现新的白色或灰色对象。
于是，在允许灰色赋值器存在的算法，最坏的情况下，
回收器只能将所有赋值器线程停止才能完成其跟对象的完整扫描，也就是我们所说的 STW。

### 8.2.4 新分配对象的颜色

新的分配过程会导致赋值器持有新分配对象的引用。可想而知我们需要为新产生的对象分配适当的颜色。
可想而知，新分配对象的颜色会产生不同的影响：

1. 如果新分配的对象为黑色或者灰色，则赋值器直接将其视为无需回收的对象，写入堆中；
2. 如果新分配的对象为白色，则可以避免无意义的新对象保留到下一个垃圾回收的周期。

如果我们进一步思考，则能够发现，由于黑色赋值器由于已经被回收器扫描过，
不会再对其进行任何扫描，一旦其分配新的白色对象
则意味着会导致错误的回收。因此黑色赋值器不能产生白色对象，
除非赋值器能够保证分配的白色对象的引用能够被写入到灰色波面中，
但这实践起来并不容易。不难看出，为了简化实现复杂度，**令新分配的对象为黑色通常是安全的。**

<!-- TODO: 新分配对象为白色？ add reference -->

## 8.2.5 赋值器屏障技术

我们在谈论垃圾回收器的写屏障时，其实是指赋值器的写屏障，即**赋值器屏障**。
赋值器屏障作为一种同步机制，使赋值器在进行指针写操作时，能够“通知”回收器，进而不会破坏弱三色不变性。

屏障上需要依赖多种操作来应对指针的插入和删除 [Pirinen, 1998]：

- 扩大波面：将白色对象作色成灰色
- 推进波面：扫描对象并将其着色为黑色
- 后退波面：将黑色对象回退到灰色

根据灰色赋值器和黑色赋值器的不同，分别会有不同类别的赋值器屏障。我们针对两种不同
类型的赋值器来分别介绍两个与 Go 有关的赋值器屏障：
灰色赋值器的 Dijkstra 插入屏障与黑色赋值器的 Yuasa 删除屏障。

### 灰色赋值器的 Dijkstra 插入屏障

**插入屏障（insertion barrier）技术**，又称为**增量更新屏障（incremental update）**[Wilson, 1992] 。
其核心思想是把赋值器对已存活的对象集合的插入行为通知给回收器，进而产生可能需要额外（重新）扫描的对象。
如果某一对象的引用被插入到已经被标记为黑色的对象中，这类屏障会**保守地**将其作为非白色存活对象，
以满足强三色不变性。

Dijkstra 插入屏障 [Dijkstra et al. 1978] 作为诸多插入屏障中的一种，
对于插入到黑色对象中的白色指针，无论其在未来是否会被赋值器删除，该屏障都会将其标记为可达（着色）。
在这种思想下，避免满足条件 1 的出现：

```go
// 灰色赋值器 Dijkstra 插入屏障
func DijkstraWritePointer(slot *unsafe.Pointer, ptr unsafe.Pointer) {
    shade(ptr)
    *slot = ptr
}
```

`shade(ptr)` 会将尚未变成灰色或黑色的指针 `ptr` 标记为灰色。通过保守的假设 `*slot` 可能会变为黑色，
并确保 `ptr` 不会在将赋值为 `*slot` 前变为白色，进而确保了强三色不变性。图 2 展示了三个对象之间，赋值器和回收器的对 ABC 对象图的操作，赋值器修改 ABC 之间的引用关系，而回收器根据引用关系进一步修改 ABC 各自的颜色。

<div class="img-center" style="margin: 0 50px 0px 30px; float: left; max-width: 50%">
<img src="../../../assets/gc-wb-dijkstra.png"/>
<strong>图 2: 使用 Dijkstra 写屏障的赋值器</strong>
</div>

Dijkstra 屏障的优势在于：

1. 性能优势：它不需要对指针进行任何处理，因为指针的读操作通常比写操作高出一个或更多数量级。
2. 前进保障：与 Steele 写屏障不同，对象可从白色到灰色单调转换为黑色，因此总工作量受到堆大小的限制。

Dijkstra 写屏障的缺点在于对性能的权衡：

但存在两个缺点：

- 由于 Dijkstra 插入屏障的保守，在一次回收过程中可能会产生一部分被染黑的垃圾对象，只有在下一个回收过程中才会被回收；
- 在标记阶段中，每次进行指针赋值操作时，都需要引入写屏障，这无疑会增加大量性能开销，为了避免造成性能问题，可以选择关闭栈上的指针写操作的 Dijkstra 屏障。当发生栈上的写操作时，将栈标记为恒灰（permagrey）的，但此举产生了灰色赋值器，将会需要标记终止阶段 STW 时对这些栈进行重新扫描。

### Dijkstra 屏障的正确性

TODO: 形式化证明

### 黑色赋值器的 Yuasa 删除屏障

**删除屏障（deletion barrier）技术**，又称为**基于起始快照的屏障（snapshot-at-the-beginning）**。
其思想是当赋值器从灰色或白色对象中删除白色指针时，通过写屏障将这一行为通知给并发执行的回收器。
这一过程很像是在操纵对象图之前对图进行了一次快照。

如果一个指针位于波面之前，则删除屏障会保守地将目标对象标记为非白色存活对象，进而避免条件 2 来满足弱三色不变性。
具体来说，Yuasa 删除屏障 [Yuasa, 1990] 对于在回收过程中，对于被赋值器删除最后一个指向这个对象导致该对象不可达的情况，
仍将其对象进行着色：

```go
// 黑色赋值器 Yuasa 屏障
func YuasaWritePointer(slot *unsafe.Pointer, ptr unsafe.Pointer) {
    shade(*slot)
    *slot = ptr
}
```

为了防止丢失从灰色对象到白色对象的路径，应该假设 *slot 可能会变为黑色，
为了确保 ptr 不会在被赋值到 *slot 前变为白色，shade(*slot) 会先将 *slot 标记为灰色，
进而该写操作总是创造了一条灰色到灰色或者灰色到白色对象的路径，进而避免了条件 2。

Yuasa 删除屏障的优势则在于不需要标记结束阶段的重新扫描，
结束时候能够准确的回收所有需要回收的白色对象。
缺陷是 Yuasa 删除屏障会拦截写操作，进而导致波面的退后，产生冗余的扫描，如图 3 所示。

<div class="img-center" style="margin: 0 30px 0px 50px; float: right; max-width: 50%">
<img src="../../../assets/gc-wb-yuasa.png"/>
<strong>图 3: 使用 Yuasa 写屏障的赋值器</strong>
</div>

### Yuasa 屏障的正确性

TODO: 形式化证明

## 混合写屏障

在诸多屏障技术中，Go 使用了 Dijkstra 与 Yuasa 屏障的结合，
即**混合写屏障（Hybrid write barrier）技术** [Clements and Hudson, 2016]。
Go 在 1.8 的时候为了简化 GC 的流程，同时减少标记终止阶段的重扫成本，
将 Dijkstra 插入屏障和 Yuasa 删除屏障进行混合，形成混合写屏障，沿用至今。

## 基本思想

该屏障提出时的基本思想是：对正在被覆盖的对象进行着色，且如果当前栈未扫描完成，
则同样对指针进行着色。

但在最终实现时原提案 [Clements and Hudson, 2016] 中对 ptr 的着色还额外包含
对执行栈的着色检查，但由于时间有限，并未完整实现过，所以混合写屏障在目前的实现是：

```go
// 混合写屏障
func HybridWritePointerSimple(slot *unsafe.Pointer, ptr unsafe.Pointer) {
	shade(*slot)
	shade(ptr)
	*slot = ptr
}
```

在 Go 1.8 之前，为了减少写屏障的成本，Go 选择没有启用栈上写操作的写屏障，
赋值器总是可以通过将一个单一的指针移动到某个已经被扫描后的栈，
从而导致某个白色对象被标记为灰色进而隐藏到黑色对象之下，进而需要对栈的重新扫描，
甚至导致栈总是灰色的，因此需要 STW。

混合写屏障为了消除栈的重扫过程，因为一旦栈被扫描变为黑色，则它会继续保持黑色，
并要求将对象分配为黑色。

混合写屏障等同于 IBM 实时 JAVA 实现中使用的 Metronome 中使用的双重写屏障。
这种情况下，垃圾回收器是增量而非并发的，但最终必须处理严格限制的世界时间的相同问题。

## 混合写屏障的正确性

直觉上来说，混合写屏障是可靠的。那么当我们需要在数学上逻辑的证明某个屏障是正确的，应该如何进行呢？

TODO：补充正确性证明的基本思想和此屏障的正确性证明

## 实现细节

TODO:

## 批量写屏障缓存

在这个 Go 1.8 的实现中，如果无条件对引用双方进行着色，自然结合了 Dijkstra 和 Yuasa 写屏障的优势，
但缺点也非常明显，因为着色成本是双倍的，而且编译器需要插入的代码也成倍增加，
随之带来的结果就是编译后的二进制文件大小也进一步增加。为了针对写屏障的性能进行优化，
Go 1.10 和 Go 1.11 中，Go 实现了批量写屏障机制。
其基本想法是将需要着色的指针统一写入一个缓存，
每当缓存满时统一对缓存中的所有 ptr 指针进行着色。

TODO:

## 小结

并发回收的屏障技术归根结底就是在利用内存写屏障来保证强三色不变性和弱三色不变性。
早期的 Go 团队实践中选择了从提出较早的 Dijkstra 插入屏障出发，
不可避免的在为了保证强三色不变性的情况下，需要对栈进行重扫。
而在后期的实践中，Go 团队提出了将 Dijkstra 和 Yuasa 屏障结合的混合屏障，
将强三色不变性进行了弱化，从而消除了对栈的重新扫描这一硬性要求，使得在未来实现全面并发 GC 成为可能。

## 许可

[Go under the hood](https://github.com/golang-design/under-the-hood) | CC-BY-NC-ND 4.0 & MIT &copy; [changkun](https://changkun.de)

---
title: "JVM固定堆大小原理概解"
layout: post
author: 曹德高
tags: [jvm]
categories: Jvm
---

### **前言**

可能很多人都知道Java程序上生产后，运维人员都会设定好JVM的堆大小，而且还是把最大最小设置成一样的值。那究竟是为什么呢？

你是否有这个疑问？设置堆大小为何要设置成两个相同的固定值，一般不是小的设置小点，大的是一个上限值，我们一般的人认知不也是说随用随取吗，你设置为大小都一样会不会一开始就把空间全占了，让自己独享经济呗。

我也有这个疑问，昨天我的同事让我给某个应用这么配，我也没想明白，然后我就找了好多资料，总结后跟你分享一下。

### **初始化内存做了什么**

一般而言，Java程序如果你不显示设定该值得话，会自动进行初始化设定。

```bash
　　-Xmx #的默认值为你当前机器最大内存的 1/4		如本机有4G内存，这里1G
　　-Xms #的默认值为你当前机器最大内存的 1/64		这里就是64M
```

显然这样配置的意义是希望JVM可以根据当前运行的环境，动态伸缩堆内存大小。之所以生产上设置成固定大小，网上也是说法有很多，很多时候都是使用“防止内存抖动”这样的模糊词语给出解释。但是我相信各位也很懵很难去理解，不知道这个词具体表达什么含义。

最大堆或最小堆，从字面上理解就是JVM在运行Java程序时，为其分配堆内存空间的上限和下限值。我们把最大和最小堆设置成相同值那意思就是分配了固定大小的内存呗。这样不就省去了动态调整内存（申请和释放）以及频繁的用户态和内核态的切换带来的开销吗？

常理推论肯定是这样的。然而当我们尝试去做个模拟实验，事实却并非如此。比如，随便写个Java程序，使用如下命令启动之。并设置好固定大小堆为1G。

```
java -Xmx1024m -Xms1024m -jar demo.jar
```

然后我们通过查看进程的内存占用时，发现程序并没有占用1G的空间，而是很小的占用。这个实验结果和我们预期的完全不一致，**并非独占、独享空间**。如下图所示。

![1](/images/java-fixed-heap-size/1-1956987.png)

**究竟是什么原因呢？**

问题其实出在我们对内存模型的理解上有问题。很多人可能都是像上面图中那样理解程序分配内存的。实际上是不对的，且也更复杂。首先我们要理解一个重要概念，那就是“进程的虚拟地址空间”，我们应用程序通过（malloc）这个系统函数申请内存，实际上就是申请了一个虚拟的内存，并不是真正的物理内存。**大家要注意，这个虚拟的内存就是指“进程的虚拟地址空间”，而不是我们通常理解的Windows下的虚拟内存或Linux下的swap(分区交换)**。

```
注：
   malloc 函数是C语言库函数，void *malloc(size_t size) 分配所需的内存空间，并返回一个指向它的指针。
   free 函数释放由其分配的内存。
```

### **内存操作真相**

应用程序申请的虚拟内存（虚拟地址空间），也就是通过malloc函数调用，本质就是在进程的虚拟地址空间里分配了一块**地址范围**而已。32位系统理论上最大4G，每个进程都有自己的虚拟地址空间，都能申请到最大4G内存。但是申请了的内存，如果没有实际使用（写入数据），**则操作系统不会给这块虚拟空间分配实际的物理内存**。其实原因很简单，物理内存一直属于紧缺资源，所以现代操作系统都设计为由内核程序统一管理，**应用程序无权直接干涉**。不是说你申请多少就真的给你多少，而是你实际使用多少才会给你多少。

你发现启动后程序内存占用很小就是这个原因。尽管JVM已经在你启动时向系统申请了1G的固定堆大小空间。程序刚启动时并没有实际的操作业务，所以你实际上只用到了很小的物理内存空间。但是如果随着系统的运行业务量越来越大，实际占用物理内存就会越来越多，直到达到申请的上限值1G。运行期间，你的程序同时也会通过GC释放一些对象，并在适当的时机归还一些物理内存给操作系统。所以占用的物理内存大小，也会动态有所调整。这样操作系统就可以给其他程序使用，提高了内存利用效率。这样的设计也没什么不好的。

![2](/images/java-fixed-heap-size/2.png)

如上图所示，操作系统对内存管理是以页为基本单位的，一个页代表了一个固定大小的地址范围。应用程序给某个变量比如String赋值时，此时该变量对应的进程虚拟地址空间所在的页在物理内存上找不到对应的页映射时，就会触发了一个缺页中断异常，操作系统就会重新将虚拟地址的页映射到物理内存中的页，此时才是真正实现了内存分配，会占用实际的物理内存空间。假如Java程序的GC把这个String变量收回了，也就是不需要占用内存空间了，用户进程的堆管理器会适当的归还一些物理内存给操作系统，以便下次可以给其他任何程序使用。**需要注意的是应用程序调用的malloc和free两个函数，都是针对应用进程的虚拟地址空间而言的，并不是实际操作物理内存。只有操作系统才拥有对实际物理内存的管理权限**。操作系统可以使用有效的各种算法，来独立高效的管理物理内存。

我们发现在实际的Java程序，配置成固定堆大小后，内存占用一旦上去了就下不来了。即使当前程序处于比较空闲的状态下。这又是为什么呢？难道Java的GC没有回收内存？

其实并不是GC没有回收内存，上面有提到GC回收内存并不是指物理内存，而是指当前进程的虚拟内存（虚拟地址空间）。一般而言，回收的虚拟内存并不会立即归还给操作系统，从而操作系统也就无法回收它了。至于何时归还物理内存，这取决于一个叫glibc的堆管理器。它根据一定的策略和算法适当的释放真实的物理内存。否则即便Java程序GC了对象，该对象占用的物理内存也不会立即释放出来。由于这里我们是设置了固定大小的堆空间，实际上GC回收的虚拟内存，也不会被释放归还给操作系统。故Java进程内存占用一旦增长，内存占用几乎都不会再下降了，这样也是出于对象再分配的效率考虑的。**这样显然可以避免操作系统反复把进程的虚拟地址页复映射物理内存页（缺页中断异常）操作，导致频繁的用户态和内核态切换造成的性能问题**。

### 总结

现在我们大概知道了为什么要设置JVM堆内存一样的原理了，总结起来有两点金句：

**1：应用进程的内存都是操作虚拟地址空间，并不是实际操作物理内存。**

**2：避免操作系统反复把进程的虚拟地址页复映射物理内存页（缺页中断异常）操作，导致频繁的用户态和内核态切换造成的性能耗损。**
---
layout: post
title: "利用内存对齐优化细微的性能"
subtitle: ' memory alignment'
author: "cslqm"
header-style: text
tags:
  - 随想
---

# 利用内存对齐优化细微的性能

学过C语言的同学都知道内存对齐这个东西。当时学时，并没有意识到原来内存对齐是可以用来优化程序性能的。

比如，golang也是类似C的对齐方案。

## 内存对齐
首先，我们知道CPU对内存读取方式并不是一位位一字节字节的访问的。CPU访问是以一块一块取的。块的大小可以为2、4、6、8、16字节等这种整倍大小。
块的大小称为内存访问粒度。这个大小一般和编译器有关，32位就是4对齐，64位就是8对齐了。


## 对齐规则
结构体的成员变量，第一个成员变量的偏移量为 0。往后的每个成员变量的对齐值必须为编译器默认对齐长度（#pragma pack(n)）或当前成员变量类型的长度（unsafe.Sizeof），取最小值作为当前类型的对齐值。其偏移量必须为对齐值的整数倍。
结构体本身，对齐值必须为编译器默认对齐长度（#pragma pack(n)）或结构体的所有成员变量类型中的最大长度，取最大数的最小整数倍作为对齐值。
结合以上两点，可得知若编译器默认对齐长度（#pragma pack(n)）超过结构体内成员变量的类型最大长度时，默认对齐长度是没有任何意义的。

## 优化举例

``` golang
type Part1 struct {
	a bool
	b int32
	c int8
	d int64
	e byte
}

type Part2 struct {
	e byte
	c int8
	a bool
	b int32
	d int64
}
```

Part1和Part2直观来看是不是都应该是1+4+1+8+1字节使用量。但是对齐之后实际内存使用情况会有巨大的变化。

先分析Part1（以4字节为块大小），由于 a bool 和 b int32 是1字节和4字节使用量所以他们放不到一个块内，所以是2个块的大小所以可以表示为axxx\|bbbb。
a和b都已经排好位置，c int8 和 d int64分别是1字节和8字节使用量所以也不能放到一个块内，所以就是3个块可以表示为cxxx\|dddd\|dddd。e byte 被剩下来了只能自己占用一个块，exxx。
所以Part1内存分布 axxx\|bbbb\|cxxx\|dddd\|dddd\|exxx,用了6个块。

在来分析Part2（以4字节为块大小），e byte 、 c int8、a bool 自身使用量都是1字节，他们可以放到一个块内，表示为 ecax。
b int32自己正好4字节，为1个块，表示为 bbbb。
d int64为8字节，为2个块，表示为dddd\|dddd。
所以Part2内存分布 ecax\|bbbb\|dddd\|dddd。

可以看到Part1比Part2占用的内存多了8个字节。

这样就完成了对程序的优化了。

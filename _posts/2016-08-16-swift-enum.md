---
layout: post
date: 2016-08-16 16:40
title: Swift 不烧脑体操 - Optional 嵌套 & Enum Associated Value
disqus: y
---

## 前言
很久以前看过巧神的一篇文章 ——  [Swift 烧脑体操（一） - Optional 的嵌套](http://blog.devtang.com/2016/02/27/swift-gym-1-nested-optional/)，最近又有道友提起了这篇文章，抱怨说好烧啊好烧啊 😂, 为什么会有两个 some 啊。其实 optional 的嵌套并不难理解，其实是很多人不清楚为什么有两个 some，因此觉得很烧脑。

## Optional

在 xcode 中我们声明一个 `Int？`类型的变量，然后用`fr v -R`打印变量的内部结构。

```swift
let a: Int? = nil
let b: Int? = 1
```
输出结果:

```swift
//(lldb) fr v -R a
(Swift.Optional<Swift.Int>) a = none {
  some = {
    _value = 0
  }
}
//(lldb) fr v -R b
(Swift.Optional<Swift.Int>) b = some {
  some = {
    _value = 1
  }
}
```
然后有人会很疑惑，为什么会多出一个 some。

先说说 `Optional`， Optional 是个枚举类型，在 Swift 中是这样定义的:

```swift
public enum Optional<Wrapped> : ExpressibleByNilLiteral {
  case none
  case some(Wrapped)
}
```
既然是个`enum`，我们就从 `enum`入手

## Enum Associated Value

Swift 的 Enum 十分强大，得益于 Swift Enum 的 Associated Value，我们可以给枚举类型添加一些额外的信息，也正是利用了 Enum 的 Associated Value 实现了 Swift 中的 Optional。我们先定义一个带 Associated Value 的枚举类型:

```swift
enum Test {
//  虽然不怎么优雅，但是能解决问题 😂
    case fuck(Int)
    case you(Double)
    case none
}

let a = Test.fuck(1)
let b = Test.none
```
然后我们继续用 `fr v -R` 查看内部结构:

```swift
//(lldb) fr v -R a
(Swift_Demo.Test) a = fuck {
  fuck = {
    _value = 1
  }
  you = {
    _value = 4.9406564584124654E-324
  }
}
//(lldb) fr v -R b
(Swift_Demo.Test) b = none {
  fuck = {
    _value = 0
  }
  you = {
    _value = 0
  }
}
```

看出来什么没？
enum 存储了关联值时，用枚举类型的名字作为存储的 key，`value` 的类型即是我们定义的 Associated Value 的类型。当一个枚举类型中的某个 `case` 有多个关联值时，则直接看成`Tuple`类型， 例如

```swift
enum Barcode {
    case UPCA(Int, Int, Int, Int)
    case QRCode(String)
}
```

那么 `Barcode` 的关联值的结构应该是这样的

```swift
UPCA: (Int, Int, Int, Int)
QRCode: String
```

## 总结
通过对 Enum 的了解，我们终于找到了原因。有两个`some`是因为`enum`使用了`case`的名称作为存储对应`Associated Value` 的 key，所以，第二个 some 是关联值的 key。想要更深入的了解可以查看 Swift 源码。

## 参考
[唐巧博客: Swift 烧脑体操（一） - Optional 的嵌套](http://blog.devtang.com/2016/02/27/swift-gym-1-nested-optional/)

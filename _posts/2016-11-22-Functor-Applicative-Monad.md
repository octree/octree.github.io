---
layout: post
date: 2016-11-22 16:45
title: Swift - Functor、Applicative & Monad
disqus: y
---

## 前言

在纯函数式语言 Haskell 中有几个十分重要的概念，Functor、Applicative 和 Monad。在刚接触 Swift 时，看到有些人通过借鉴 Haskell 中的这些特性写出一些十分风骚的代码，十分羡慕，自己也想学习一下这些东西。刚开始是通过一些博客去了解 Monad 的，但是看来看去总是迷迷糊糊。索性就学习了一下 Haskell，通过 Haskell 了解了 `Functor` `Applicative` `Monad` 的概念。才算对 Monad 有了一些了解。不过稍微深入学习一点，就都成了数学问题，本文并不会对范畴论的东西做讨论，毕竟自己还是一知半解，还要努力学习。

## Functor

在 Haskell 中，Functor 是这样定义的：

```haskell
class Functor (f :: * -> *) where
  fmap :: (a -> b) -> f a -> f b
  // ....
```

在 Haskell 中，class 用来声明  `typeclass`（类似 Swift 中的接口），`f :: * -> *` 代表一种数据类型，并且这种类型是对另一种类型的封装（而且只能是一种，多一个少一个，都不能算一种 🙃 ）。例如 Swift 中的 Optional：
 
```swift
enum Optional <T> {
    case some(T)
    case none
}
```

`Optional` 类型是对任意类型 `T` 的封装，所以，Optional 是可以实现 `Functor`

再举一个 🌰，那就是我们网络封装中常用的枚举 `Result``：

```swift
enum Result<T> {
    case success(T)
    case failure(Error)
}
```

Result 是对类型 T 的封装，所以 Result 也可以有 Functor。

重新回到 Functor 的定义，实现 `Functor` 首先要实现一个 `fmap` 函数，这个函数在 `Haskell` 中的类型是`(a -> b) -> f a -> f b`
如果对应到 `Result` 转换成 Swift 的定义就是这样子的：

```swift
/// 假装 Swift 3.0 还支持柯里化
func fmap<T, U>(_ f:(T) -> U) (r: Result<T>) -> Result<U>
```

显然这样是不美观的，所以，还是要根据 Swift 的特性，修改下

```swift
extension Result<T> {
    
       func fmap(f: (T) -> U) -> Result<U>  
}

```

这就变成了我们常见`map 函数`的样子。

根据定义可以看出，fmap 是对所封装的值的变换，所以，实现就变成了这样子：

```swift

extension Result<T> {
    
       func map<U>(_ f:(T) -> U) -> Result<U> {
          switch self {
          case .success(let v):
            
             return .success(f(v))
          case .failure(let e):
             return .failure(e)
         }
       }
}

```

`Functor` 需要满足一下两个规则，在 `Haskell` 中是这样描述的：

```haskell
fmap id == id  -- 1
fmap (f.g) = fmap f (fmap g)  -- 2
```

id 是一个函数，swift 实现可以看成这样:

```swift
func id<T>(x:T) -> T {
    return x;
}
```
第一条规则表示，如果把 `id` 传给 `fmap` 得到的结果应该和变换前是一样的，所以，如果你的 Result 本来是 `.success` 转换后变成 `.failure` 了，这就不是 `Functor` 了，反之亦然。

第二个表示：`result.fmap(g).fmap(f)` 和 `result.fmap({ v in f(g(x)) })` 得到的结果应该是相同的。

## Applicative

首先先看一下 `Applicative` 在 Haskell 中的定义

```haskell
class Functor f => Applicative (f :: * -> *) where
  pure :: a -> f a
  (<*>) :: f (a -> b) -> f a -> f b
 // ....
```

 `Applicative` 定义是 `class Functor f => Applicative (f :: * -> *)` 意味着 `Applicative` 首先也要是个 `Functor`，`f :: * -> *` 这里和 `Functor` 类似。

接下来我们改成 Swift 实现：

```swift
extension Result<T> {

    static func pure(x: T) -> Result<T> {
        return .success(x)
    }
    
    //  <*>
    func apply<U>(_ ap:Result<(T) -> U>) -> Result<U> {
        switch ap {
        case .success(let f):
            return self.fmap(f)
        case .failure(let e):
            return .failure(e)
        }
    }
}
```

可以看出，`Applicative` 是一个被封装的函数应用到另一个被封装的值上。Applicative 需要满足四个规则：

```haskell
pure id <*> v ≡ v   -- 1
pure (.)  <*> u <*> v <*> w == u <*> (v <*> w)  -- 2
pure f <*> pure x ≡ pure (f x) 
u <*> pure y ≡ pure ($ y) <*> u -- 4
```

1. 和 functor 的第一条规则类似。
2. 第二个是 `Applicative` 的结合律, `.` 是一个函数，用 swift 可以这样实现：

   ```swift
    // .
    func compose<T, U, V>(_ f: (U) -> V, g:(T) -> U) -> (T) -> V {
        return { f(g($0)) }
    }
    //  f.g ≡ compose(f, g)
   ```
3. `$` 用 Swift 可以这样实现：

   ```swift
    // $
    func apply<T, U>(f:(T) -> U, v: T) -> U {
        return f(v)
    }
    // f $ v ≡ apply(f:f,v:v)
   ```

## Monad

同样，还是先看 `Monad` 在 `Haskell` 中的定义：

```haskell
class Applicative m => Monad (m :: * -> *) where
  (>>=) :: m a -> (a -> m b) -> m b
  return :: a -> m a
```

`class Applicative m => Monad (m :: * -> *) ` 意味着 `Monad` 首先也是一个 `Applicative`。`m :: * -> *` 这里和 `Functor` 类似，可以认为这种类型是另一种类型的封装。

`>>=` 对应对应 Swift 中常用的 `flatMap`。我们首先把这种类型中定义的方法修改成 `Result` 枚举中的方法。
`return` 函数了 pure 类似。

```swift
extension Result<T> {

    static func `return`(x: T) -> Result<T> {
        return .success(x)
    }
    
    func flatMap<U>(f: (T) -> Result<U>) -> Result<U> {
        
        switch self {
        case .success(let v):
            
            return f(v)
        case .failure(let e):
            return .failure(e)
        }
    }
}
```

`Monad` 需要满足三个规则：

```haskell
 m >>= return  ≡ m         // 1
 return x >>=  f  ≡ f x      // 2
 m >>= f >>= g  ≡ m >>= \x -> f x >>= g    // 3
```

1. 和 `Functor` 的规则 1 类似
2. 就是把 `f` 作为 `flatMap` 调用和直接把封装的值用来调用 `f` 函数得到的结果应该是相同的。
3. 翻译成 Swift 代码就是这样子的

```swift
let result = ....
let a = result.flatMap(f).flatmap(g) 
let b = result.flatMap {
    f($0).flatMap(g)
}
```

`a` 和 `b` 得到的结果也是相同的。

## 应用

Functor 与 Monad 的应用还是非常常见的，例如使用 `map` 或者 `flatMap` 代替 `if let` 绑定。初次之外，还有就是把异步调用转换成 `then{..}.then{..}.error{..}` 的形式，类似 javascript 中的 Promise 库。本篇就不详细解释了。

## 总结
1. 本文中的函子、单子的规则来自 Haskell，略不同于数学意义上的函子，严格来说，本文介绍的函子都是自函子。关于范畴论上的函子，参考下一篇文章。

哈哈哈哈~~~








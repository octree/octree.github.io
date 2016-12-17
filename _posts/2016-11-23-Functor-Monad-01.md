# 
---
layout: post
date: 2016-11-23 12:51
title: Functor & Monad - 范畴论
disqus: y
---

## 前言
先前写了一篇对 Functor 和 Monad 的简单介绍，思前想后，还是写一点对于 Functor 和 Monad 的更深一点的理解。本文章算是对自己学习的笔记，如果能够帮助别人一下也是极好的，哈哈哈~~~

## 范畴

范畴在数学中代表一堆数学实体和实体之间的关系。一个范畴 C 应该包含以下 3 个要素：
1. 一个集合类 ob(C)，其元素被称为`物体`
2. 物体间的态射构成的集合 home(C)，每个态射都只有一个源物体 a 和一个目标物体 b，且 a 和 b 都在 ob(C) 中，态射 f 被称为从 a 至 b 的态射。记作 f : a -> b
3. 对任三个物件a、b和c，二元运算hom(a, b)×hom(b, c)→hom(a, c)称之为态射复合；f : a → b 和 g : b → c 的复合写成 g o f，并且需要满足一下两个公理：
  * 单位率：对于任意物体 x，总存在一个态射 idx: x -> x, 使得对于每一个态射 f : a -> b 都会有 idb o f  = f o ida
  * 结合律：若 f : a -> b、g : b -> c 以及 h : c -> d，则 h o ( g o f ) = ( h o g ) o f

  举一个范畴的例子：



## 函子（Functor）与 范畴


## 自函子（EndoFunctor）

## 半群 （Semigroup）& 单位半群（Monoid）

## 单子（Monad）

## 总结

1. 半成品，有时间再写

## 参考

- 《魔力 Haskell》
- 《范畴论》
- [我所理解的 Monad](http://hongjiang.info/understand-monad-0/)
-  [维基百科-范畴](https://zh.wikipedia.org/wiki/%E7%AF%84%E7%96%87_(%E6%95%B8%E5%AD%B8))
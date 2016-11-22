---
layout: post
date: 2016-11-22 19:04
title: 2016-11-22 19:04
disqus: y
---

## mod & rem

`mod` 取模，向负无穷取值
`rem` 取余，向 0 取值

```haskell
mod -4 3  -- 2
rem -4 3   -- -1
```

## TODO
1. WHNF & NF
2. nonstrict evali

## quot & div

```haskell
(quot x y) * y + (rem x y) == x
(div x y) * y + (mod x y) == x
```

## CheatSheet

- `:`  cons
- `head` take first elm
- `tail` second to end
-  `last` last elm
- `init` 0 - (n-1)
- `take`
    ```haskell
    take 2 "hello" --"he"
    ``` 
- `drop`
    ```haskell
    drop 2 "hello" -- "llo"
    drop 2 [1, 2, 3] -- [3]
    ```
- `!!`
    ```haskell
    "hello" !! 1 -- 'e'
    [1, 2, 3] !! 2 -- 3
    ```

- `minimum` list 中最小
- `maximum` 
- `sum` 计算 list 所有元素的和
- `product` 所有元素的积
- `elem` 判断一个元素是否包含在一个 list 中
- `cycle` 接收一个 list，返回一个无限循环的 list
- `repeat` 接收一个值，返回一个无限循环的 list
- `replicate 3 10` -> [10, 10, 10]
- `odd` 判断是否为奇数
- `even` 判断偶数
- 

## List

```haskell
-- [expr | cond [, cond] ]

[x * 2 | x <- [1 .. 10] ,x < 5]
-- [2, 4, 6, 8]
```
 
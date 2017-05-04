---
layout: post
date: 2017-04-01
title: Objective-C 实现一个 JSON 解释器
disqus: y
---

## 前言

以前有一篇文章介绍了如何使用 Swift 实现一个解释器，在文章的结尾提到用 Swift 实现了一个 JSON 的切词器，但是效率很低，后来发现，原因是 Swift 的字符串处理的效率低到令人发指。所以本来打算写一篇用 Swift 解析 JSON 的文章也就作罢了，改成使用 Objective-C + C 语言实现。

## 词法分析器

### Token 类型
和使用用 Swift 写词法分析器一样，先确定有那些 Token 类型

```C
OCTJSONTokenType enum _OCTJSONTokenType {
    OCTJSONTokenTypeNumber              =           0,          //    数字
    OCTJSONTokenTypeString                   =           1,          //    字符串
    OCTJSONTokenTypeBoolean               =           2,          //    Bool
    OCTJSONTokenTypeLeftBracket          =           3,          //    [
    OCTJSONTokenTypeRightBracket        =           4,          //    ]
    OCTJSONTokenTypeLeftBrace              =           5,          //    {
    OCTJSONTokenTypeRightBrace            =           6,          //    }
    OCTJSONTokenTypeColon                    =           7,          //    :
    OCTJSONTokenTypeComma                 =           8,          //    ,
    OCTJSONTokenTypeEof                         =           9,          //    ]
} OCTJSONTokenType;
```

### 铁路图

通过语法图能够辅助我们进行编码。首先对画出不同 Token 类型的铁路图




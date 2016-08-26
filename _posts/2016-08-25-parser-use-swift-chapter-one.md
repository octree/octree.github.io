---
layout: post
date: 2016-08-25 18:40
title: 用 Swift 实现简单的解释器 (一)
disqus: y
---

## 前言

今年早些时候听过一次傅若愚的分享，是关于怎么把 `String`解析成 `Float`的，然后引申到了 JSON 的解析。 那个时候我还不了解编译原理，听得云里雾里，他叫若愚，我当时感觉自己可能是“若智”。 从那个时候我就想要自己写一个`JSON Parser`。然后就学习了下编译原理的知识。
在学习过程中自己也写了几个简单的`Parser`，都是用 `Swift` 实现的。本篇文章主要是分享自己是怎么解释算数表达式的，可以解释带`+` `-` `*` `/` `括号`的算式 。
## 什么是解释器

> 解释器是能够把高级编程语言一行一行解释运行的程序

解释器会把某种高级语言的代码转换成另一种形式运行，与编译器不同的时，解释器不会把源程序转换成机器代码。

## 解释步骤

问: 把大象放进冰箱需要几步？不对，把解释代码需要几步？
答: 三步

![](http://obb77efas.bkt.clouddn.com/intepreter.jpg)

我们的源代码是一长串的字符串，首先要对这串字符串进行分割，把代码分割成多个单词。
例如

```swift
1 + 2
```
会被分割成 `1`  `+`  `2` 三个 Token，Token 中保存了 token 的类型和 value。把代码分割成 Token 的过程叫`词法分析`，进行分割的这段代码程序称之为`Lexer`(词法分析器)

然后再通过词法分析器把 Token 转换成 `AST` (抽象语法树)，这个过程叫做`语法分析`，接下来你肯定知道了，进行语法分析的程序叫做`Parser`(语法分析器)

然后解析器通过遍历 `AST` 运行程序

## 词法分析器

首先我们要明确 Token 的类型:

```
//  1 + 2 - 3 * ( 4 * 5 + ( 6 ) )
public enum TokenType {
    
    case integer    //  整数
    case plus        //  加号
    case minus    //  减号
    case mul       //  乘号
    case div       //  除号
    case leftParen    //  左括号
    case rightParen   //  右括号
    case eof            //  文件结尾
}
```

然后定义 `Token` 的结构:

```swift
public struct Token {
    
    public let type: TokenType
    public let value: Any
}
```

我们可以选择使用正则表达式来进行切词，或者自己写一个 `Lexer` 这里我选择了后者，由于水平有限，效率并没有正则表达式的高 😂。 `Lexer` 代码如下:

```swift
public class Lexer {
    
    private let text: String
    private var currentChar: Character? = nil // 当前字符
    private var pos = 0                 //  当前扫描位置
    
    public init(text: String) {
        
        self.text = text
        if text.characters.count > 0 {
            
            currentChar = text[pos]
        }
    }
    
    public func nextToken() -> Token {
        
        skipWhiteSpace()
        if let ch = currentChar {
            
            if ch.isDigit {
                
                return Token(type: .integer, value: integer())
            } else if ch == "+" {
                
                successor()
                return Token(type: .plus, value: ch)
            } else if ch == "-" {
                
                successor()
                return Token(type: .minus, value: ch)
            } else if ch == "*" {
                
                successor()
                return Token(type: .mul, value: ch)
            } else if ch == "/" {
                
                successor()
                return Token(type: .div, value: ch)
            } else if ch == "(" {
                
                successor()
                return Token(type: .leftParen, value: ch)
            } else if ch == ")" {
                
                successor()
                return Token(type: .rightParen, value: ch)
            }
            
            fatalError("parsing failed")
        }
        
        return Token(type: .eof, value: None)
    }
    
    //  integer token scanner
    
    private func integer() -> Int {
        
        var result = 0
        
        while let ch = currentChar, ch.isDigit {
            
            successor()
            result *= 10
            result += Int(String(ch))!
        }
        
        return result
    }

    // 移动扫描位置
    private func successor() {
        
        pos += 1
        if pos >= text.characters.count {
            
            currentChar = nil
        } else {
            
            currentChar = text[pos]
        }
    }
    
    // 过滤空格
    private func skipWhiteSpace() {
        
        while let ch = currentChar, ch == " " {
            
            successor()
        }
    }
}
```

Ok，一个简单的 `Lexer` 就这样完成了，测试一下:

```swift
let lexer = Lexer(text: " 2  * (  3 + 1)")
lexer.nextToken()    //  type: integer
lexer.nextToken()    //  type: mul
lexer.nextToken()    //  type: leftParen
lexer.nextToken()    //  type: integer
lexer.nextToken()    //  type: plus
lexer.nextToken()    //  type: integer
lexer.nextToken()    //  type: rightParen
lexer.nextToken()    //  type: eof
```

## 语法解析器

语法解析器是把`Token`转换成`AST`，在转换之前，我们们必须要知道要把什么样的`Token`序列要处理成怎样的结构。
举个栗子🌰:

1 + 2 + 3: 的 `AST` 应该是这样的:

![AST 1](http://obb77efas.bkt.clouddn.com/ast001.jpg?imageView2/2/w/300)

再举一个栗子:

1 + 2 * 3 的抽象语法树 :

![AST 1](http://obb77efas.bkt.clouddn.com/ast002.jpg?imageView2/2/w/300)

我们可以通过 `BNF` 或者`语法图`来表示语法，这里为了更直观的表现，我使用了语法图

### 语法图

首先我们画一个整数加法的语法图 ( 我的丑逼字体 ) :

![](http://obb77efas.bkt.clouddn.com/parser-1/sytax-diagram-plus.jpg?imageView2/2/w/300)

语法图中的路径即是整数加法表达式的需要满足的`Token`序列。

我们今天要解析的算式的的语法图是这样:

![](http://obb77efas.bkt.clouddn.com/parser-1/sytax-diagram-factor.jpg?imageView2/2/w/300)

![](http://obb77efas.bkt.clouddn.com/parser-1/syntax-diagram-term.jpg?imageView2/2/w/300)

![](http://obb77efas.bkt.clouddn.com/parser-1/syntax-diagram-expr.jpg?imageView2/2/w/300)

### Parser 实现

通过语法图我们能够直观的看出`Token`序列，下面要做的就是把语法图转换成代码.
因为要把`Token`转化为`抽象语法树`，我们先要定义`二叉树`

```swift
public class AST {
    
    let token: Token
    init(token: Token) {
        
        self.token = token
    }
}

//  + - * / 双目运算符
public class BinOp: AST {
    
    let left: AST
    let right: AST
    
    public init(token: Token, left: AST, right: AST) {
        
        self.left = left
        self.right = right
        super.init(token: token)
    }
}
//  数字
public class Number: AST {
    
    public var intValue: Int {
        
        return token.value as! Int
    }
}
```

然后根据`语法图`实现 `Parser` 代码:

```swift
import Foundation

//  语法分析器
public class Parser {
    
    private let lexer: Lexer
    private var currentToken: Token
    
    public init(text: String) {
        
        lexer = Lexer(text: text)
        currentToken = lexer.nextToken()
    }
    
    private func eat(type: TokenType) {
        
        if currentToken.type == type {
            
            currentToken = lexer.nextToken()
        } else {
            
            fatalError("unexpected token type")
        }
    }
    
    private func factor() -> AST {
        
        let token = currentToken
        if token.type == .integer {
            
            eat(type: .integer)
            return Number(token: token)
        } else if token.type == .leftParen {
            
            eat(type: .leftParen)
            let node = expr()
            eat(type: .rightParen)
            return node
        }
        
        fatalError("Parser: unexpected factor")
    }

    // term      
    private func term() -> AST {
        
        var node = factor()
        while [TokenType.mul, .div].contains(currentToken.type) {
            
            let token = self.currentToken
            if currentToken.type == .mul {
                
                eat(type: .mul)
            } else {
                
                eat(type: .div)
            }
            node = BinOp(token: token, left: node, right: factor())
        }
        
        return node
    }
    
    private func expr() -> AST {
        
        var node = self.term()
        while [TokenType.plus, .minus].contains(currentToken.type) {
            
            let token = currentToken
            if currentToken.type == .plus {
                
                eat(type: .plus)
            } else {
                
                eat(type: .minus)
            }
            node = BinOp(token: token, left: node, right: term())
        }
        
        return node
    }
    
    public func parse() -> AST {
        
        return expr()
    }
}

```

然后验证一下:

```swift
let parser = Parser(text: "(1 + 2) + 2 * (3 + 4)")
//  笔者写的一个用来画二叉树的 View，用来直观的检验结果
TreeCanvasView(tree: parser.parse())
```
我们写的 Parser 把`(1 + 2) + 2 * (3 + 4)`转化的语法抽象树如下图所示:
![](http://obb77efas.bkt.clouddn.com/parser-1/parser-tree.jpg?imageView2/2/w/400)
十分符合预期 😂
## 解释器

通过我们的不懈努力，终于把 token 转化成了语法抽象树。接下来的事情就简单的多了，只需要遍历`AST`得出运算结果就好了。

```swift
import Foundation


//  解释器
public class Interpreter {
    
    private let parser: Parser
    public init(text: String) {
        
        parser = Parser(text: text)
    }
    
    public func interpret() -> Int {
        
        let tree = parser.parse()
        return visit(tree: tree)
    }
    
    func visit(tree: AST) -> Int {
        
        if let number = tree as? Number {
            
            return visitNumber(tree: number)
        } else if let op = tree as? BinOp {
            
            return visitBinOp(tree: op)
        }
        
        return 0
    }
    
    
    func visitNumber(tree: Number) -> Int {
        
        return tree.intValue
    }
    
    func visitBinOp(tree: BinOp) -> Int {
        
        switch tree.op.type {
        case .plus:
            
            return visit(tree: tree.left) + visit(tree: tree.right)
        case .minus:
            
            return visit(tree: tree.left) - visit(tree: tree.right)
        case .mul:
            
            return visit(tree: tree.left) * visit(tree: tree.right)
        case .div:
            
            return visit(tree: tree.left) / visit(tree: tree.right)
        default:
            fatalError("bug is coming")
        }
    }
}
```

测试一下我们的解释器

```swift
let interpreter0 = Interpreter(text: "1 ")
interpreter0.interpret()    //    1
let interpreter1 = Interpreter(text: "1 + 2")
interpreter1.interpret()    //    3
let interpreter2 = Interpreter(text: "1 + 2 * (3 + 4) + 4 / 2 ")
interpreter2.interpret()    //    17
```

## ToDo

1. 接下来完成设计一个简单的脚本语言，完成该脚本语言的解释器
2. 我用这种方法写了一个 `JSON Parser`, 但是字符太多的时候效率低得惊人，学习怎么优化，等效率稍微不这么低写个文章分享下。

## 参考

- 编译原理
- 两周自制脚本语言
- [https://ruslanspivak.com/](https://ruslanspivak.com/)


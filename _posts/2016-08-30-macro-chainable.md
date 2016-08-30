---
layout: post
date: 2016-08-30 12:10
title: 自动生成 Objective-C 链式调用代码
disqus: y
---

## 前言

最近公司小伙伴用`Objective-C`写了一个表单控件，但是配置表单依然需要写很多代码，特别是配置表单的 Model，这个 model 有很多属性(title，isRequired 等)，在设置 model 属性的地方需要写很多代码，然后就想可不可以像`Masonry`一样，搞个链式调用，少写几行代码。

## 链式调用

首先说一下如何实现链式调用，首先，我们知道链式调用的代码是这样的

```objectivec
obj.title(@"hello").isRequired(YES);
```

在 `Objective-C` 中什么可以用 `.` 调用，是不是无参的实例方法是不是可以直接用 `.` 调用。然后什么类型可以在后面加 `(...)` 执行某些代码，是不是马上想到了`block`，最后为了完成链式调用，需要执行的`block`返回实例本身。所以链式调用可以这么写:

```objectivec
@class Test;
typedef Test *(^OCTNSStringChainable)(NSString *string);
typedef Test *(^OCTBOOLChainable)(BOOL flag);

@interface Test : NSObject
@property (nonatomic, copy) NSString *title;
@property (nonatomic, getter=isRequired) BOOL required;
    
- (OCTBOOLChainable)oct_required;
- (OCTNSStringChainable)oct_title;
@end

@implementation Test

- (OCTBOOLChainable)oct_required {

    __weak __typeof(self) wself = self;
    return ^Test* (BOOL flag) {
        
        __strong __typeof(wself) sself = wself;
        
        if(sself) {
        
            sself.required = flag;
        }
        
        return self;
    };
}
    
- (OCTNSStringChainable)oct_title {

    __weak __typeof(self) wself = self;
    return ^Test* (NSString *string) {
        
        __strong __typeof(wself) sself = wself;
        
        if(sself) {
            
            sself.title = string;
        }
        
        return self;
    };
}
```

然后就可以通过链式调用的方法设置属性:

```objectivec
test.oct_title(@"title").oct_required(YES);
```

## 子类

在笔者的实际项目中，刚开始提到的 model，有一个基类，然后还有各种继承该基类的子类，我对基类做了链式调用，但是子类如果直接继承基类的方法，在调用`oct_title(@"...")` 后返回的类型是基类的类型，后面的链式调用就只能调用基类中提供的方法。我们要想一种方法让子类中的`oct_title` 返回的 block 返回子类的类型。但是定义`OCTBOOLChainable`的时候，返回类型已经确定了，如果要实现上述功能，需要我们在子类中重新定义各种`Chainable`的 `block` ，需要写大量的重复代码。笔者就想，能不能让代码自动生成，减少一点工作量。

## 自动生成代码

在自动代码的生成中，笔者只使用了`宏`，如果你对宏还不了解可以先去猫神的博客了解一下[宏定义的黑魔法](https://onevcat.com/2014/01/black-magic-in-macro/)

自动生成链式调用代码的代码是这样的:

```objectivec
// 1
#define __chain_block_name(x,y) OCT##x##y##Chainable

// 2
#define declare_block_no_asterisk(x, y) \
    typedef x* (^ __chain_block_name(x,y))(y value);
// 3
#define declare_block_asterisk(x, y)\
    typedef x* (^ __chain_block_name(x,y))(y * value);
// 4
#define auto_declare_block(x) \
    @class x;\
    declare_block_no_asterisk(x, NSUInteger)\
    declare_block_no_asterisk(x, double)\
    declare_block_no_asterisk(x, CGRect)\
    declare_block_no_asterisk(x, CGSize)\
    declare_block_no_asterisk(x, NSTimeInterval)\
    declare_block_no_asterisk(x, CGPoint)\
    declare_block_no_asterisk(x, CGFloat)\
    declare_block_no_asterisk(x, NSInteger)\
    declare_block_no_asterisk(x, BOOL)\
    declare_block_no_asterisk(x, id)\
    declare_block_asterisk(x, NSString)\
    declare_block_asterisk(x, NSMutableString)\
    declare_block_asterisk(x, NSNumber)\
    declare_block_asterisk(x, NSArray)\
    declare_block_asterisk(x, NSMutableArray)\
    declare_block_asterisk(x, NSDictionary)\
    declare_block_asterisk(x, NSMutableDictionary)\
    declare_block_asterisk(x, NSURL)\
    declare_block_asterisk(x, NSData)\
    declare_block_asterisk(x, NSDate)\
    declare_block_asterisk(x, NSMutableData)

// 5
#define declare_method(x,y,z) \
- (__chain_block_name(x,y))oct_##z;

// 6
#define imp_method_no_asterisk(x,y,z) \
- (__chain_block_name(x,y))oct_##z {\
    __weak typeof(self) wself = self;\
    return ^x *(y value) {\
        __strong typeof(self) sself = wself;\
        if (sself) {\
            sself.z = value;\
        }\
        return self;\
    };\
}

// 7
#define imp_method_asterisk(x,y,z) \
- (__chain_block_name(x,y))oct_##z {\
    __weak typeof(self) wself = self;\
    return ^x *(y * value) {\
        __strong typeof(self) sself = wself;\
        if (sself) {\
             sself.z = value;\
        }\
        return self;\
    };\
}
```

1. `__chain_block_name`: 用来生成 `Chainable Block` 的名字
2. `declare_block_no_asterisk`: 用来定义`Chainable Block`，但是类型后面不跟 *，例如`NSInteger`、`NSTimeInterval`、`id` 等。
3. `declare_block_asterisk`: 用来定义`Chainable Block`，但是类型后面跟 *，例如`NSString`、`NSDate` 等
4. `auto_declare_block`: 定义默认类型的`Chainable Block`
5. `declare_method`:  声明实例方法，它返回一个`Chainable Block`。
6. `imp_method_no_asterisk`: 实现实例方法，与`declare_method` 对应，返回使用`declare_block_no_asterisk`定义的`block`
7. `imp_method_asterisk`: 实现实例方法，与`declare_method` 对应，返回使用`declare_block_asterisk`定义的`block`

### 使用方法:

```objectivec
typedef void (^OCTCustomBlock) (NSInteger num);

auto_declare_block(Test)
declare_block_no_asterisk(Test, OCTCustomBlock)

@interface Test : NSObject

@property (nonatomic) NSInteger num;
@property (copy, nonatomic) NSString *string;
@property (copy, nonatomic) OCTCustomBlock code;

declare_method(Test, NSString, string)
declare_method(Test, NSInteger, num)
declare_method(Test, OCTCustomBlock, code)

@end

@implementation Test

imp_method_no_asterisk(Test, NSInteger, num)
imp_method_asterisk(Test, NSString, string)
imp_method_no_asterisk(Test, OCTCustomBlock, code)
    
- (NSString *)description {

    return [NSString stringWithFormat:@"test: %zd %@", self.num, self.string];
}
@end
```

## 总结

缺陷:

1. 只能用于修改 `readwrite` 类型的 property 的值
2. 生成的代码不利于调试

[Demo 地址](https://github.com/octree/OCTMacro)



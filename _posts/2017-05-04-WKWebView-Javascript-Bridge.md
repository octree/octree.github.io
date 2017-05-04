---
layout: post
date: 2017-05-04
title: WKWebView —— Web 与 Native 交互
disqus: y
---

## 前言

在 App 中，有些复杂的界面用 Native 并不容易实现，所以有时候我们会选择使用 Web 技术实现一些界面，把 Web 界面嵌入到 Native App 中，使用 WebView 渲染界面。这些 Web 界面有时候需要使用到 Native 中的一些数据，抑或需要向 Native 反馈一些信息，这时候就需要接触到 Native 和 Web 交互的一些知识。

在 iOS 8 之前，iOS 开发都是使用 UIWebView 来渲染 Web 页面，UIWebView 可以直接使用 `JavascriptCore` 来实现交互。但是 UIWebView 存在很严重的内存泄漏的问题。iOS 8 苹果推出了 WKWebView 来代替 UIWebView。WKWebView 和 Safari 浏览器使用了相同的 Javascript 引擎。据苹果说，WKWebView 更加流畅，具有高效的 App 与 Web 信息交换通道。当然，所以如果你的 App 只支持 iOS 8.0 之后的系统，你完全可以使用更好用的 WKWebView 来渲染你 App 内的 Web 页面。

具体 WKWebView 有哪些 API，怎么使用，这里就不做介绍。主要还是说一下 Native 与 Web 交互的问题。

## WKScriptMessageHandler

在 Web 页面中，你可以直接通过一下代码给 Native 发送消息

```javascript
// { Name }: Message Name
window.webkit.messageHandlers.{NAME}.postMessage()
```

在 Native 代码中，你只需要实现 `WKScriptMessageHandler` 协议，并且添加到 WKWebView 的 `userContentController` 中，就可以接收 Web 页面发送来的消息了。
当然，这些只是我们要用到的技术，还需要封装一下，让 Native 和 Web 交互更加简单易用。

## 整体架构

![img](http://obb77efas.bkt.clouddn.com/wkwebviewbridge_arch.png)

预想的设计如上图

* Plugins: 负责处理 Web 的调用，例如：跳转 Native 页面、修改 Navigationbar 标题、获取一些 JSON 数据等。
* Injector: 负责向 Web 页面注册 plugins, 让 Web 通过调用自定义的函数就可以向 Native 发送消息，或者从 Native 获取信息。
* Injector - Dispacth: 负责分发 Web 页面传过来的消息，根据消息调用特定的 Plugin。
*  message： 负责向 Native 发送消息
* callback dispatch: cache 负责缓存一些回调函数(当 web 需要从 native 获取数据的时候)，并且当 native 处理完成后，把数据传回到 web 时，负责调用 callback 函数。

## Message & Callback Dispatch

message 部分负责从 Web 向 Native 发送消息，这部分代码我们使用 javascript 完成，通过 WKWebView 注入到 Web 页面。callback dispatch 部分负责回调函数的缓存和调用。

```javascript
//   core

if (!window.bridge) {

    window.bridge = {}
}

window.bridge.callback = {
    index: 0,
    cache: {
    },
    invoke: function(id, args) {
        let key = '' + id
        let callback = window.bridge.callback.cache[key]
        callback(args)
//        delete window.bridge.callback.cache[key]  // 1
    }
}

window.bridge.invoke = function(id, selector, callback, ...args) {
    let index = -1
    if (callback !== null && callback !== undefined ) {
        window.bridge.callback.index += 1
        index = window.bridge.callback.index
        window.bridge.callback.cache['' + index] = callback
    }
    window.webkit.messageHandlers.bridge.postMessage({"identifier": id, "selector": selector, "callbackId": index, "args": args })
}
```
1. 刚开始这个地方是 Web 的回调函数执行一次就从 cache 中删除，但是考虑到可能需要执行多次(例如: 不断获取手机的地理位置、加速度等)。又懒得分成两种情况，干脆都留着。

## Plugin
Plugin 就是用来处理 Web 向 Native 发送的消息。首先要定义一种类协议，规定 plugin 应该提供哪些信息。

```objectivec
typedef NS_ENUM(NSInteger, OCTWebViewPluginInjectionTime) {

    OCTWebViewPluginInjectionTimeAtDocumentStart,
    OCTWebViewPluginInjectionTimeAtDocumentEnd,
};

/**
 *  JS 函数需要 Native 处理数据后，回调传回数据
 *
 *  @param params js 回调函数的参数
 */
typedef void (^OCTResponseCallback)(NSDictionary *params);

/**
 *  WebView 的扩展，把 Native Bridge 到 WebView 中的 Object 需要实现这个协议
 */

@protocol OCTWebViewPlugin <NSObject>

@optional

@property (nonatomic, readonly) OCTWebViewPluginInjectionTime injectionTime;

@required
/**
 *  唯一标识
 */
@property (copy, nonatomic, readonly) NSString *identifier;
/**
 *  javascript 代码，主要作用是来定义在 webview 中，javascript 函数的名称
 */
@property (copy, nonatomic, readonly) NSString *javascriptCode;

@end

```

这里可以先忽略，先知道有这么一个协议就行，后面会给出具体使用的例子。

## Injector

Injector 负责注册自定义的插件，同时负责调把 web 的消息分发到插件中执行。

### Injector 的缓存与销毁
首先需要一个类用来缓存 WKWebView 的 Injector，使每一个 WKWebView 的实例都由一个唯一的 Injector。这个上图貌似忘了画了，蛤蛤。

```objectivec
@interface OCTWebViewInjectorCache()

@property (strong, nonatomic) NSMutableDictionary *injectorMap;

@end

@implementation OCTWebViewInjectorCache

#pragma mark - Life Cycle

+ (instancetype)sharedInstance {
    static OCTWebViewInjectorCache *manager;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        manager = [[OCTWebViewInjectorCache alloc] initPrivacy];
    });
    return manager;
}

- (instancetype)initPrivacy {
    
    if (self = [super init]) {
        
    }
    return self;
}


#pragma mark - Public Method

- (OCTWebViewPluginInjector *)injectorForWebView:(WKWebView *)webView {
    
    if (!webView) {
        return nil;
    }
    //  1    
    NSString *key = [NSString stringWithFormat:@"%p", webView];
    OCTWebViewPluginInjector *injector = self.injectorMap[key];
    if (!injector) {
        injector = [[OCTWebViewPluginInjector alloc] initWithWebView:webView];
        self.injectorMap[key] = injector;
    }
    return injector;
}


- (void)removeInjectorForWebView:(WKWebView *)webView {
    
    NSString *key = [NSString stringWithFormat:@"%p", webView];
    [self.injectorMap removeObjectForKey:key];
}

#pragma mark - Accessor

- (NSMutableDictionary *)injectorMap {
    
    if (!_injectorMap) {
        
        _injectorMap = [NSMutableDictionary dictionary];
    }
    return _injectorMap;
}

@end

```

1. 为了不对 WKWebView 的实例增加多余的引用，又起到对应关系，这里使用指针地址作为 key

当 WKWebView 销毁的时候，我们还希望与之对应的 Injector 一起销毁掉。

```objectivec
@interface WKWebView (_OCTWebViewInjector)

@end

@implementation WKWebView (_OCTWebViewInjector)

+ (void)load {
    
    method_exchangeImplementations(class_getInstanceMethod(self.class, NSSelectorFromString(@"dealloc")), class_getInstanceMethod(self.class, @selector(swizzled_dealloc)));
}

- (void)swizzled_dealloc {
    
    [[OCTWebViewInjectorCache sharedInstance] removeInjectorForWebView:self];
}
@end
```

### Injector 初始化

Injector 初始化的时候我们需要向 WKWebView 中注入上文中  `Message & Callback Dispatch` 的 javscript 代码，为 plugins 提供必要的 api 支持。

```objectivec
- (void)injectCorePlugin {

    [self injectJavascriptCode:self.messageJavascriptCode injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:NO];
}

- (void)injectJavascriptCode:(NSString *)jscode injectionTime:(WKUserScriptInjectionTime)time forMainFrameOnly:(BOOL)flag {
    
    WKUserScript *script = [[WKUserScript alloc] initWithSource:jscode injectionTime:time forMainFrameOnly:flag];
    [self.webView.configuration.userContentController addUserScript:script];
}
```

### Injector 分发消息

接收 Web 的消息只需要实现 `WKScriptMessageHandler` 就可以了。当 Web 向 Native 发送消息的时候，就会调用 `userContentController:didReceiveScriptMessage:` 方法。下方代码用来分发消息到具体的 plugin 中执行。如果 Web 需要 Native 执行完成后返回一些数据，还会处理一下 callback 的事务，触发 Web 执行 callback 函数。

```objectivec
- (void)invokeWithCommands:(NSDictionary *)commands {
    
    NSString *identifier = commands[kOCTMessageIdentifierKey];
    if (!identifier) {
        return;
    }
    
    id obj = self.pluginMap[identifier];
    if (!obj) {
        return;
    }
    
    id selector = commands[kOCTMessageSelectorKey];
    if (!selector || ![selector isKindOfClass:[NSString class]]) {
        
        return;
    }
    
    id args = commands[kOCTMessageArgsKey];
    if (![args isKindOfClass:[NSArray class]]) {
        return;
    }
    
    NSInteger callbackId = [commands[kOCTMessageCallbackIdKey] integerValue];
    [self invokeWithBridgeObj:obj selector:selector args:args callbackId:callbackId];
}

- (void)invokeWithBridgeObj:(id<OCTWebViewPlugin>)obj selector:(NSString *)selector args:(NSArray *)args callbackId:(NSInteger)callbackId {
    
    SEL sel = NSSelectorFromString(selector);
    NSMethodSignature *signature = [[obj class] instanceMethodSignatureForSelector:sel];
    
    if (!signature) {
        NSLog(@"%@ has no selector: %@", NSStringFromClass([obj class]), selector);
        return;
    }
    
    NSInteger selectorArgsCount = [signature numberOfArguments] - 2;
    
    NSInteger jsArgsCount = args.count + (callbackId >= 0 ? 1: 0);
    
    NSString *lastObjType = [NSString stringWithCString:[signature getArgumentTypeAtIndex:selectorArgsCount + 1] encoding:NSUTF8StringEncoding];
    //   参考 type encoding
    if (![lastObjType isEqualToString:@"@?"] && callbackId > 0) {
        
        NSLog(@"JS Invoke - class: %@, selector: %@, js need callback，but this fucking selector does not support", NSStringFromClass([obj class]), selector);
        return;
    }
    
    if (jsArgsCount != selectorArgsCount) {
        
        NSLog(@"JS Invoke - class: %@, selector: %@，args count not match", NSStringFromClass([obj class]), selector);
        return;
    }
    
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:signature];
    invocation.target = obj;
    invocation.selector = sel;
    
    NSInteger index = 2;
    for (id obj in args) {
        
        [invocation setArgument:(void *)&obj atIndex:index];
        index++;
    }
    
    if (callbackId >= 0) {
        
        __weak __typeof(self) wself = self;
        
        NSString *identifier = [[self class] generateCallbackIdentifier];
        OCTResponseCallback callback = ^void(NSDictionary *param) {
            __strong __typeof(wself) sself = wself;
            [sself invokeCallbackWithId:callbackId param:param];
            
            [self.callbackMap removeObjectForKey:identifier];
        };
        self.callbackMap[identifier] = callback;
        [invocation setArgument:(void *)&callback atIndex:index];
    }
    
    [invocation invoke];
}

- (void)invokeCallbackWithId:(NSInteger)callbackId param:(NSDictionary *)param {
    
    NSString *json = @"null";
    if (param) {
        NSError *error;
        NSData *data = [NSJSONSerialization dataWithJSONObject:param options:NSJSONWritingPrettyPrinted error:&error];
        if (!error) {
            json = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
        }
    }
    
    NSString *code = [NSString stringWithFormat:@"window.bridge.callback.invoke(%zd, %@)", callbackId, json];
    [self.webView evaluateJavaScript:code completionHandler:nil];
}

#pragma mark - WKScriptMessageHandler

- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message {
   
    if (![message.name isEqualToString:kOCTMessageName]) {
        
        return;
    }
    
    if ([message.body isKindOfClass:[NSDictionary class]]) {
        
        [self invokeWithCommands:message.body];
    }
}
```

### Inject Plugin

```objectivec
- (void)injectPlugin:(id<OCTWebViewPlugin>)plugin {
    
    NSParameterAssert(plugin.identifier != nil);
    NSParameterAssert(plugin.javascriptCode != nil);
    
    self.pluginMap[plugin.identifier] = plugin;
    [self injectJavascriptCode:plugin.javascriptCode
                 injectionTime:[self userScriptInjectionTimeForPlugin:plugin]
              forMainFrameOnly:NO];
}
```

Injector 使用 plugin 的 identifier 为 key， plugin 为 value 缓存注入的 plugins。这里的 plugin 的 javascriptcode 属性依然可以忽略，后面会有解释。

## 使用 Injector

### 栗子1 —— 在 xcode 输出 web 页面的 log 信息

我们希望有这个一个函数，我们通过调用它就能把信息打印到 xcode 的 Output 控制台中。首先要实现 `OCTWebViewPlugin` 协议。

```objectivec
@interface OCTLogPlugin : NSObject <OCTWebViewPlugin>
@end

@implementation OCTLogPlugin

- (NSString *)identifier {
    
    return @"me.octree.bridge.log";
}

- (NSString *)javascriptCode {
    // 1
    NSString *path = [[NSBundle bundleForClass:[self class]] pathForResource:@"log" ofType:@"js"];
    return [NSString stringWithContentsOfFile:path encoding:NSUTF8StringEncoding error:NULL];
}
- (void)log:(id)msg {
    NSLog(@"WebView Bridge: %@", [msg description]);
}
@end
```

1. 代码中使用的 javascript 代码：

   ```javascript
   window.bridge.log = function(msg) {
       window.bridge.invoke("me.octree.bridge.log", "log:", null, msg)
   }
   ```
invoke 的四个个参数分别对应了：plugin 的 identifier， plugin 的 selector， callback 函数和参数

web 开发者只需要调用` window.bridge.log('fuck')` 就可以把执行 Native 的 Log 函数。

## 还可以更简单些

为了更简单的实现交互，通过实现一个通用的 `BlockPlugin` 让开发者可以直接使用 `block` 就能实现 Native 和 Web 的交互。

```objectivec
@interface OCTBlockPlugin : NSObject <OCTWebViewPlugin>
- (instancetype)initWithFunctionPath:(NSString *)path handler:(void(^)(NSDictionary *data))block;
- (instancetype)initWithFunctionPath:(NSString *)path handlerWithResponseBlock:(void(^)(NSDictionary *data, OCTResponseCallback responseCallback))block;
@end

@interface OCTBlockPlugin ()

@property (strong, nonatomic) void (^handler)(NSDictionary *param);
@property (strong, nonatomic) void (^handlerWithResponseBlock)(NSDictionary *param, OCTResponseCallback block);

@end

@implementation OCTBlockPlugin

#pragma mark - Life Cycle

- (instancetype)initWithFunctionPath:(NSString *)path handler:(void(^)(NSDictionary *data))block {
    if (self = [super init]) {
        
        NSParameterAssert(path.length > 0);
        NSParameterAssert(block != nil);
        _handler = block;
        _identifier = [NSString stringWithFormat:@"me.octree.plugin3.%@", path];
        _javascriptCode = OCTJavascriptCodeForPath(path, _identifier, NO);
    }
    return self;
}

- (instancetype)initWithFunctionPath:(NSString *)path handlerWithResponseBlock:(void(^)(NSDictionary *data, OCTResponseCallback responseCallback))block {
    if (self = [super init]) {
        
        NSParameterAssert(path.length > 0);
        NSParameterAssert(block != nil);
        self.handlerWithResponseBlock = block;
        _identifier = [NSString stringWithFormat:@"me.octree.plugin4.%@", path];
        _javascriptCode = OCTJavascriptCodeForPath(path, _identifier, YES);
    }
    
    return self;
}


#pragma mark - Private Method

- (void)invoke:(NSDictionary *)param {

    !self.handler ?: self.handler(param);
}

- (void)invoke:(NSDictionary *)param callback:(OCTResponseCallback)callback {

    !self.handlerWithResponseBlock ?: self.handlerWithResponseBlock(param, callback);
}
@end
```

创建一个 plugin 的成本大大降低，当然，代价就是会有一定的局限性，就是 Block 的类型比较单一，双方通过 `json` 互相传递数据。使用方法如下：

```objectivec
[[OCTWebViewPluginInjector injectorForWebView:_webView] injectPluginWithFunctionPath:@"network.fetchData" handlerWithResponseBlock:^(NSDictionary *data, OCTResponseCallback responseCallback) {
            NSLog(@"%@", data);
            responseCallback(@{@"yooo": @"ssss"});
}];
```

web 调用：

```javascript
network.fetchData({name: 'octree'}, json => {
    //  do something
})
```

### 更多

初次之外，通过这种方式还可以实现更多的功能，例如注入 css 代码修改 web 页面的样式，实现通用的夜间模式等等。这里就不一一介绍了。我写了一个 demo，里面有很多例子，希望一起探讨。

[demo 地址](https://github.com/octree/OCTWebViewBridge)



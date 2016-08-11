---
layout: post
date: 2016-08-01 12:56
disqus: y
title: '获取 iOS 设备上安装的应用列表'
---

##前言
  在虾神的公众号中提到，iOS 8 以后可以通过`MobileCoreService` 的私有 API获取 iOS 设备上安装的所有应用。
代码如下：
  
  ```objectivec
Class cls = NSClassFromString(@"LSApplicationWorkspace");
id s = [(id)cls performSelector:NSSelectorFromString(@"defaultWorkspace")];
NSArray *arr = [s performSelector:NSSelectorFromString(@"allInstalledApplications")];
  ```  
`arr`的元素的类型是 `LSApplicationProxy`,该类的具体信息可以从 [LSApplicationProxy.h](https://github.com/nst/iOS-Runtime-Headers/blob/master/Frameworks/MobileCoreServices.framework/LSApplicationProxy.h) 获取。该类包含了应用名称、版本等信息。
然后，笔者想通过这个方法写一个类似 iOS 桌面的东西玩玩。本文总结了开发过程中遇到的一些问题以及解决方法。

##获取 App 的图标
###方法一
获取 icon 的本地存储的位置，然后通过 `imageWithContentsOfFile:` 方法获取 icon，代码如下：

```objectivec
NSString *path = [[item performSelector:NSSelectorFromString(@"resourcesDirectoryURL")] path];
NSString *iconName = [[[boundIconsDictionary objectForKey:@"CFBundlePrimaryIcon"]
                           objectForKey:@"CFBundleIconFiles"] lastObject];
NSString *iconPath = [NSString stringWithFormat:@"%@/%@.png", path, iconName];
UIImage *iconImage = [UIImage imageWithContentsOfFile:iconPath];
```
通过这种方法可以获取到 App icon 的 filepath，但是在界面上只能显示系统 App 的图标，由于沙盒的限制，并不能获取到第三方 App 的图标。

###方法二
`LSApplicationProxy` 中有一个方法：`- (id)iconDataForVariant:(int)arg1;`，通过这个方法貌似可以获取 icon data, 于是我尝试使用这个 Data 实例化一个 UIImage 对象。

```objectivec
NSData *data = [item performSelector:NSSelectorFromString(@"iconDataForVariant:") withObject:@(2)];
UIImage *image = [UIImage imageWithData:data];
```
很遗憾，失败了。
朕寝食难安。于是把微信 icon 的 NSData 的内容 log 出来，观察一下，log 信息如下
>2000afa7 20000000 57000000 57000000 60010000 08000000 20000000 02200000 0dd500ff 0dd500ff 0dd500ff 0dd500ff 0dd500ff 0dd500ff 0dd500ff 0dd500ff 0dd500ff 0dd500ff 0dd500ff ···

通过观察不同 App 的图标的 NSData 信息，笔者发现，前 32 个字节基本相同。其次我们知道，一个像素的 RBGA 数据可以存储在一个 4 个字节的空间中，例如上图中的 `0dd500ff` 表示某种绿色。从 log 信息中，还发现，前 32 个字节之后的数据对应的颜色只包含绿色和白色，恰好和微信的图标吻合。

通过观察和验证，前 32 个字节中的第 9 和第 13 个字节分别存储了图片的宽度和高度，而通过 `- (id)iconDataForVariant:(int)arg1;` 参数为 2 时获取的图片的宽度高度都是 87，所以，我这里偷懒直接用了 87。

先来检验一下我的猜想。

```objectivec
NSInteger lenth = self.iconData.length;
NSInteger width = 87;
NSInteger height = 87;
uint32_t *pixels = (uint32_t *)malloc(width * height * sizeof(uint32_t));
[self.iconData getBytes:pixels range:NSMakeRange(32, lenth - 32)];

CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();

//  注意此处的 bytesPerRow 多加了 4 个字节
CGContextRef ctx = CGBitmapContextCreate(pixels, width, height, 8, (width + 1) * sizeof(uint32_t), colorSpace, kCGBitmapByteOrder32Little | kCGImageAlphaPremultipliedFirst);
    
CGImageRef cgImage = CGBitmapContextCreateImage(ctx);
CGContextRelease(ctx);
CGColorSpaceRelease(colorSpace);
UIImage *icon = [UIImage imageWithCGImage: cgImage];
CGImageRelease(cgImage);
    
return icon;
```
通过这种方法成功获得了所有 App 的 icon，界面如下

![](http://obb77efas.bkt.clouddn.com/IMG_1115.PNG?imageView2/2/w/300)

为什么 `bytesPerRow` 多加了 4 个字节呢？，从第 32 个字节之后的数据对应了图片上每一行的像素，而每行数据的最后位置添加了`00000000`进行隔离，如果`bytePerRow`直接使用 87 会导致第 i 行像素偏移 i 个位置（注：i 起始为 0），所以，多加了 4 个字节是为了过滤掉用来分割的 `00000000`

###方法三
还有个最简单地方法，itunes 提供了一个 API: `http://itunes.apple.com/lookup?id={appId}` 可以通过 id 获取应用的信息，会返回一段 JSON 数据，这个方法就不多做介绍了。

##App 跳转
实现 App 跳转就比较简单了，笔者在 [LSApplicationWorkspace.h](https://github.com/nst/iOS-Runtime-Headers/blob/master/Frameworks/MobileCoreServices.framework/LSApplicationWorkspace.h)  中找到了这样一个方法：
`- (BOOL)openApplicationWithBundleID:(id)arg1;`，可以直接通过这个方法打开其他的 App，代码如下:

```objectivec
NSString *bundleIdentifier = [item performSelector:NSSelectorFromString(@"bundleIdentifier")];
Class cls = NSClassFromString(@"LSApplicationWorkspace");
id s = [(id)cls performSelector:NSSelectorFromString(@"defaultWorkspace")];
[s performSelector:NSSelectorFromString(@"openApplicationWithBundleID:") withObject:bundleIdentifier];
``` 

##最后
对于前 32 个字节，除了宽度和高度，我还不知道其它几个字节分别代表了什么意思，希望有个大神解惑。

[Demo 地址](https://github.com/octree/SpringboardDemo)

##参考
- [虾神公众号](http://mp.weixin.qq.com/s?__biz=MzIwMTYzMzcwOQ==&mid=2650948457&idx=1&sn=9b1f3e0c405b7621a6b3b7c8419a2695&scene=4#wechat_redirect)
- [iOS-Runtime-Headers](https://github.com/nst/iOS-Runtime-Headers)
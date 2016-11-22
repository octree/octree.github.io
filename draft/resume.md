
---
layout: post
date: 
title: 
disqus: y
---

## 个人信息
姓名：刘宏志
性别：男
手机：159-0661-5974
邮箱：octree@octree.me
Github: https://github.com/octree
Blog: http://octree.me

## 工作经历

### 杭州尘埃科技有限公司

- 时间：2015.04 - 2016.06
- 职位：iOS 工程师
- 工作描述：
  1. 负责云集浏览器、Link、Pero iOS 客户端的架构、维护、开发。
  2. 负责性能调优与单元测试

###  杭州迪火科技有限公司

- 时间：2016.7 - 至今
- 职位：iOS 工程师
- 工作描述：
    1. 负责火掌柜客户端的开发
    2. 参与重构，通用组件的开发。 

## 项目经历

### 云集浏览器

1.  实现基于 UIScrollView 的云桌面（类似与锤子桌面），由于需要最多同时显示 64 个 Web App，遇到了性能问题，通过使用 Instrument 进行分析，把 FPS 从 34 帧 提升到了 60 帧。
2.  实现浏览器对 React-Native 的支持，通过 method swizzling 修改 ReactNativeSDK 中 JavascriptLoader 的方法，实现加载加密过的 JSBundle 并进行解密，然后渲染界面。 
3.  通过 URL 拦截和 JS 去脚本两种方法实现网页去除广告的功能。 
4.  通过 URL 拦截获取下载信息，使用 NSURLSessionDataTask、NSOutputStream 实现断点下载。
5.  使用 CoreData 实现对数据的持久化存储。
6. 使用 DKNightVersion 实现夜间模式。
7. 使用 XCTest 框架进行单元测试。

### Link

Link 是一款类似与 WorkFlow 的应用。此应用的 iOS 端是我独立开发的。主要工作如下： 

1. 设计云任务、Action 的数据结构，设计执行 Action 与后端交互的接口。 
2. 通过事件驱动调度 Action 的执行。 
3. 使用 Instrument 工具进行性能调优。
4. 使用 AutoLayout  进行屏幕适配。
5. 使用长连接技术实现与后台的数据交互，接受任务执行成功的回调。


### Pero

Pero 是一个图片交易 App。此项目的 iOS 客户端是我独立开发的。在这个项目中使用了 Swift 语言。 

1. 实现图片上传功能，使用 GCD 实现对多张图片异步上传的监控。
2. 使用 Realm 框架实现对数据的持久化存储。
3. 使用 YYWebImage 实现对图片的异步加载与本地缓存，使用 Core Graphics 框架通过常见算法修改每个像素的值实现简单的滤镜。 
4. 项目使用 MVVM 的架构模式，结合 POP 的编程思想，进行代码解耦。
5. 集成 Alipay SDK 实现支付功能。


### 火掌柜

1. 参与重构，负责通用组件的开发。
2. TDFHTTPDNS: 应用内进行 DNS 解析。

### 其他项目
1. Pero Web 版，主要是用来上传图片和发布图包的，是一个基于 Vue.js 的单页应用。
2. YYMessage：iOS 消息提示。

##技能
1. 熟悉 Objective-C、Swift 语言。 
2. 熟悉 iOS 内存管理机制以及 ARC 技术。 
3. 熟悉 SQLite 数据库, 熟悉 XML/JSON 等数据格式，熟悉网络编程。 
4. 熟悉 iOS 开发中常用设计模式熟悉 GCD/NSOperation 等并发编程技术。 
5. 熟悉 Git 版本管理工具。熟悉 XCTest、Core Animation、Core Graphics 等框架 。
6. 能熟练运用 Instrument 工具进行性能调优。
7. 常用工具：Charles、LLDB、Postman、Debugserver、dumpdecrypted 等。


---
layout: post
date: 2015-10-17 16:05
title: 'iOS 多个异步任务的监控'
disqus: y
---

在发起网络请求时，我们一般会用异步请求，这里我们以 `AFNetWorking` 为例:

```objectivec
AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager];
[manager GET:@"http://octree.me/" parameters:nil success:^(AFHTTPRequestOperation *operation, id responseObject) {
    NSLog(@"Success");
} failure:^(AFHTTPRequestOperation *operation, NSError *error) {
    NSLog(@"Error: %@", error);
}];
```

假如我们要执行多个异步请求，一般可以这么写

```objectivec
NSArray *urlStrings = @[ @"http://octree.me", @"http://google.com", @"http://github.com" ];

for(NSString urlString in urlStrings) {
    AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager];
    [manager GET:urlString parameters:nil success:^(AFHTTPRequestOperation *operation, id responseObject) {
        NSLog(@"Success");
     } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
        NSLog(@"Error: %@", error);
}];
}

```

假如我们需要在三个请求都完成或者失败后进行一些处理，但是`manager`发起的请求时异步处理的，也就是说当`manager`调用`GET:parameters:success:failure`会立即返回，当请求成功或失败后才会调用各自的`block`，我们如何才能监控并发的异步事件呢？并且他们的完成顺序以及完成时间都是不确定的。

当然，你可以用一组`bool`值或者其他的标识记录每个任务的完成进度，但是这样的代码不仅丑陋，而且失去了扩展性。

这时候我们就可以用到 `dispatch_group` 了

`dispatch_group`在任务组内的任务都完成的时候通过同步或者异步的方式通知你

`dispatch_group` 提供了两种通知方式，`dispatch_group_wait`和`dispatch_group_notify`

`dispatch_group_wait`会阻塞当前线程，知道任务都完成时才会继续执行下面的代码

我们可以使用`dispatch_group_wait`这样实现：

```objectivec
NSArray *urlStrings = @[ @"http://octree.me", @"http://google.com", @"http://github.com" ];

dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{

	dispatch_group_t requestGroup = dispatch_group_create();

	for(NSString urlString in urlStrings) {
    dispatch_group_enter(requestGroup);
    AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager];
    [manager GET:urlString parameters:nil success:^(AFHTTPRequestOperation *operation, id responseObject) {

        NSLog(@"Success");
        dispatch_group_leave(requestGroup);
     } failure:^(AFHTTPRequestOperation *operation, NSError *error) {

        NSLog(@"Error: %@", error);
        dispatch_group_leave(requestGroup);
	}];
	}

    dispatch_group_wait(requestGroup, DISPATCH_TIME_FOREVER);
    dispatch_async(dispatch_get_main_queue(), ^{
            //doSomething;
        });
}];
```

因为`dispatch_group_wait`会阻塞当前线程，所以我们要把他放到后台执行，避免阻塞主线程。通过`dispatch_group_enter`和`dispatch_group_leave`手动通知任务的开始以及结束。`dispatch_group_wait`会阻塞当前线程，知道所有任务完成或者超时才会继续执行下面的代码。


前面我们说过，`dispatch_group`提供了两种通知方式，我们已经了解了`dispatch_group_wait`，另一种是`dispath_group_notify`，这种方式相对于前面的显得更为灵活。

`dispatch_group_notify`是通过异步的方式通知，所以，不会阻塞线程。于是，我们就可以这样写:

```objectivec
NSArray *urlStrings = @[ @"http://octree.me", @"http://google.com", @"http://github.com" ];

dispatch_group_t requestGroup = dispatch_group_create();

for(NSString urlString in urlStrings) {
    dispatch_group_enter(requestGroup);
    AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager];
    [manager GET:urlString parameters:nil success:^(AFHTTPRequestOperation *operation, id responseObject) {

        NSLog(@"Success");
        dispatch_group_leave(requestGroup);
     } failure:^(AFHTTPRequestOperation *operation, NSError *error) {

        NSLog(@"Error: %@", error);
        dispatch_group_leave(requestGroup);
	}];
}

dispatch_group_notify(requestGroup, dispatch_get_main_queue(), ^{

        //doSomething
    });
```
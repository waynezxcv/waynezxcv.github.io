---
layout:     post
title:      "Runloop应用举例"
subtitle:   "总结一下自己在项目中实际使用到runloop的情况。"
date:       2015-08-08
author:     "waynezxcv"
header-img: "img/post.png"
tags:
    - iOS
---


## 观察runloop状态，来执行相关操作。

### 使用场景

iOS应用中，UIScrollView及其子类（UITableView，UICollectionView等）的滚动流畅性，是影响用户体验的重要指标。而iOS中，渲染UI的操作只能在主线程中执行，所以一旦主线程出现了特别耗时的任务时，就会导致界面卡顿。我们会把比较耗时的任务（比如下载图片）等操作放到后台线程去执行，但有时还是会出现卡顿的现象。Facebook的开源框架AsyncDisplayKit的做法是把这些操作尽量推后，在“合适的”的时候去执行。这个“合适”的时候指的是当主线程在保证UI流畅性的情况下，空闲、有余力处理其他任务的时候。这时，我们可以通过观察主线程runloop的状态，来寻找这个“合适”的timing。

### TIPS

在CoreFoundation框架中，CFRunLoopObserverRef是观察者，能够监听RunLoop的状态改变，可以监听到的时间点有以下几个：

```
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),//即将要进入一个runloop循环
    kCFRunLoopBeforeTimers = (1UL << 1),//即将要处理Timer
    kCFRunLoopBeforeSources = (1UL << 2),//即将要处理Source
    kCFRunLoopBeforeWaiting = (1UL << 5),//即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),//从休眠中唤醒
    kCFRunLoopExit = (1UL << 7),//即将结束一个runloop循环
    kCFRunLoopAllActivities = 0x0FFFFFFFU//上面的所有状态
};
```

### 应用举例

在iOS当中，主线程是UI线程，有些操作只能在主线程中来完成。在facebook开源框架AsyncdisplayKit中，会把这些只能在主线程中执行的事务（如给CALayer设置contents）放在一个全局队列当中，当mainRunloop进入kCFRunLoopBeforeWaiting和kCFRunLoopExit状态时，才开始执行。


```
//注册一个runloop观察者	
static CFRunLoopObserverRef observer;
    CFRunLoopObserverContext context = {
        0,           // version
        (__bridge void *)@"call back",  // userInfo，自定义的一些信息
        &CFRetain,   // retain
        &CFRelease,  // release
        NULL         // copyDescription
    };
    observer = CFRunLoopObserverCreate(NULL,
                                       kCFRunLoopBeforeWaiting | kCFRunLoopExit,
                                       YES,
                                       INT_MAX,
                                       &_transactionGroupRunLoopObserverCallback,
                                       &context);
    CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
    CFRelease(observer);

//当runloop到达这个timing的时候，执行回调函数
static void _transactionGroupRunLoopObserverCallback(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void* info) {
    //do something...
}
```
如果我们需要在runloop的某个特定timing执行某些操作（比如被唤醒时，点击事件之前等），都可以通过添加runloop观察者的方法来实现。

***

## 常驻线程，让子线程不自动释放

### 使用场景

有时我们需要经常在后台处理耗时操作，我们不希望这个子线程在执行完毕后就被销毁掉，而保持常驻状态，以减少频繁创建和销毁线程带来的性能损耗。这时，我们就可以在这个线程当中启动一个runloop，来保持这个子线程一直到需要的时候才销毁。


### TIPS

每条线程都有唯一的一个与之对应的RunLoop对象。主线程的RunLoop已经自动创建好了，子线程的RunLoop需要主动创建。子线程调currentrunloop的时候第一次是没有的，他会先判断是否存在，没有的话会先创建。RunLoop在第一次获取时创建，在线程结束时销毁。

###应用举例

在SDWebImage中，SDWebImageDownloaderOperation类负责下载图片，在start子线程发起网络请求，然后通过NSURLConnectionDataDelegate的“- (void)connection:(NSURLConnection *)connection didReceiveResponse:(NSURLResponse *)response;”等回调方法来接收下载的数据。这时，如果不在相应的子线程中启动一个runloop，线程会在执行完请求的动作之后就被销毁，那么就永远也收不到回调了。
SDWebImage的做法是，在请求开始的后，执行"CFRunLoopRun();"方法来在子线程中启动一个runloop，这样这个线程就被保持常驻状态,正常的执行回调方法。然后在回调结束或者取消请求的操作里面执行“CFRunLoopStop(CFRunLoopGetCurrent());”来结束这个runloop，让相应的子线程被回收。在新版本的SDWebImage中，已经放弃使用NSURLConnection来进行网络请求，而使用全新的NSURLSession来进行网络请求。NSURLSession是使用Block来进行回调的，所以已经取消了线程常驻的操作。

***

## NSTimer

### 使用场景

当使用“+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti target:(id)aTarget selector:(SEL)aSelector userInfo:(id)userInfo repeats:(BOOL)yesOrNo;”方法注册一个NSTimer时，发现定时器的计时会出现误差。这是因为，我们在使用“scheduledTimerWithTimeInterval”方法时，NSTimer实例是被加到当前runloop中的，模式是NSDefaultRunLoopMode。而“当前runloop”就是应用程序的main runloop，此mainRunloop负责了所有的主线程事件，这其中包括了UI界面的各种事件。当主线程中进行复杂的运算，或者进行UI界面操作时，由于在main runloop中NSTimer是同步交付的被“阻塞”，而模式也有可能会改变。因此，就会导致NSTimer计时出现延误。

### TIPS

NSTimer的原理：当我们设置NSTimer时，需要把他注册到一个runloop当中，并给其指定一种模式。Runloop每次循环时都会检测这个timer，看是否可以触发。
在CoreFoundation框架中，系统默认注册了5个Mode:
kCFRunLoopDefaultMode：App的默认模式，通常主线程是在这个Mode下运行
UITrackingRunLoopMode：界面跟踪模式，当scrollView滑动式会切换到这个模式，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他模式影响。
kCFRunLoopCommonModes: 这是一个占位用的模式，不是一种真正的模式,它包含了kCFRunLoopDefaultMode，UITrackingRunLoopMode
UIInitializationRunLoopMode: 在刚启动 App 时第进入的第一个模式，启动完成后就不再使用
GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode，通常用不到。

### 应用举例

解决这种误差有两种办法：

* 使用下面的API


```
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];

```

并给这个定时器设置模式为NSRunLoopCommonModes。
我们可以测试一下，在界面中创建一个scrollView，然后在主线程runloop中注册一个NSTimer，代码如下：

```

- (void)viewDidLoad {
UIScrollView* scrollView = [[UIScrollView alloc] initWithFrame:self.view.bounds];
scrollView.contentSize = CGSizeMake(self.view.bounds.size.width,self.view.bounds.size.height * 2.0f);//让scrollView可以滚动起来
[self.view addSubview:self.scrollView];
NSTimer* timer = [NSTimer timerWithTimeInterval:0.1f target:self selector:@selector(timerFired:) userInfo:nil repeats:YES];

}

- (void)timerFire:(id)sender {
	NSLog(@"fired~!");
}

```

再将这个timer添加到mainRunloop的不同模式下，会产生不同的结果。


```
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];//defaultMode..一旦RunLoop进入其他模式，这个定时器就不会工作，所以滑动scrollview时，RunLoop进入UITrackingRunLoopMode，定时器就不打印了。
[[NSRunLoop currentRunLoop] addTimer:timer forMode:UITrackingRunLoopMode];//只在trackingMode下定时器工作，只有滑动scrollview时，定时器才打印。
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];//CommonMode包含上面两种mode。所以无论是否滑动scrollView可以一直打印。

```

* 在子线程注册定时器，并把timer添加到子线程的runloop当中。


```
dispatch_async(dispatch_get_global_queue(0, 0), ^{
	NSTimer* timer = [NSTimer timerWithTimeInterval:0.1f target:self selector:@selector(timerFire:) userInfo:nil repeats:YES];
	[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];//由于处理scrollview滑动是在mainrunloop，所以子线程的runloop并没有进入UITrackingRunLoopMode，定时器仍然工作，所以滑动时，仍然可以打印。
	[[NSRunLoop currentRunLoop] addTimer:timer forMode:UITrackingRunLoopMode];//由于处理scrollview滑动是在mainrunloop，Runloop并没有进入UITrackingRunLoopMode，定时器仍然没有工作，所以一直没有打印。
	[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];//可以一直打印
	[[NSRunLoop currentRunLoop] run];
});

```

>我们还可以使用GCD的定时器。GCD中除了主要的Dispatch Queue之外，还有不太引人注意的Dispatch Source。它是BSD系内核惯有功能kqueue的包装。kqueue是在XNU内核中发生各种事件时，在应用程序编程方执行处理的技术，其CPU符合非常小，尽量不占用资源。Dispatch Source与Dispatch Queue不同，是可以取消的。取消时必须执行的处理可指定为回调用的Block形式。而且一般的NSTimer定时器因为受到RunLoop影响,会存在时间不准时的情况。


CCD定时器使用方法


```
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_main_queue());
    dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 0.3f * NSEC_PER_SEC, DISPATCH_TIME_FOREVER * NSEC_PER_SEC);
    dispatch_source_set_event_handler(timer, ^{
        //timer handler
    });
    dispatch_resume(timer);//启动定时器
	dispatch_source_cancel(timer);//取消定时器
    dispatch_source_set_cancel_handler(timer, ^{
    	//do something
    });//取消定时器回调
    
```

***

## PerformSelector

### 使用场景

有时我们需要在特定的模式下去执行某些方法。

### 应用举例

使用下面API

```
performSelector:withObject:afterDelay:inModes:
performSelecter:afterDelay: 
performSelector:onThread: 

```

### TIPS

当调用 NSObject 的

```
 performSelecter:afterDelay: 
 
 ```
 
后，实际上其内部会创建一个Timer并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

当调用

```
performSelector:onThread: 

```

时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。

***



## AutoreleasePool

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

也就是说，在主线程一个Runloop循环下，自动释放池创建并释放了两次。

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

***



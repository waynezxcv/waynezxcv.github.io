---
layout:     post
title:      "GCD使用方法全方位解析"
subtitle:   "Grand Central Dispatch的详细使用方法及分析。"
date:       2015-05-05
author:     "waynezxcv"
header-img: "img/post.png"
tags:
    - Objective-C
    - GCD
---

### Grand Central Dispatch

GCD是iOS当中执行异步任务的技术之一。使用GCD无需管理线程，这由内核来完成，开发者只需要管理queue。

### GCD的API



#### dispatch_async和dispatch_sync函数
dispatch_async指异步执行，即追加第二个参数中的执行（Block中的执行）在第一个参数的队列中后，函数立即返回，不会阻塞当前线程。
dispatch_sync指同步执行，即追加第二个参数中的执行（Block中的执行）在第一个参数的队列中后，函数不会立即返回，它会一直阻塞当前线程到Block中的执行全部结束之后，才执行后面的操作。

🌰例如在主线程中执行这个操作，会造成死锁

```
    dispatch_sync(dispatch_get_main_queue(), ^{
        NSLog(@"111");
    });
    
``` 
   
因为执行dispatch_sync函数，会停止当前线程，并等待在主线程中执行的“NSLog(@"111");”执行完毕。但，当前线程就是主线程，已经停止了，所以Block中的执行永远都不会完成，造成死锁。
    

#### dispatch_set_target_queue(dispatch_object_t object, dispatch_queue_t queue);
将第一个参数的queue的优先级设置为跟第二个queue一样。
第一个参数只能是自己创建的queue，而不是通过get_global_queue或者_get_main_queue获取到的。

🌰


```

     dispatch_set_target_queue(dispatch_queue_create("aQueue", DISPATCH_QUEUE_CONCURRENT), dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0));
     
     
```

#### Dispath Group

想在追加到dispatch_queue的多个处理全部结束后执行某个结束处理，这种情况经常出现。如果只使用一个serail dispatch queue时，只要在queue最后追加这个处理就可以了。但如果使用并行队列，或者多个队列时，就会变得很复杂,这时候应该使用Dispatch Group。

🌰

```
     dispatch_queue_t concurrentQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
     dispatch_group_t group = dispatch_group_create();
     
     dispatch_group_async(group, concurrentQueue, ^{
     NSLog(@"1");
     });
     dispatch_group_async(group, concurrentQueue, ^{
     NSLog(@"2");
     });
     dispatch_group_async(group, concurrentQueue, ^{
     NSLog(@"3");
     });
     dispatch_group_async(group, concurrentQueue, ^{
     NSLog(@"4");
     });
     dispatch_group_async(group, concurrentQueue, ^{
     NSLog(@"5");
     });
     
     dispatch_group_notify(group, dispatch_get_main_queue(), ^{
     NSLog(@"end");
     });//这个函数可以检测group中的事务什么时候执行结束，一旦结束，就会把block中的事务追加queue中
     //监控第一个参数的group执行结束后，会把第三个参数中追加的事务，添加到第二个参数的queue中。
     
     long result = dispatch_group_wait(group, 1 * NSEC_PER_SEC);
     //这个函数会等待在第二个参数的时间后，来验证group是否执行完成，如果是0，表示都已经执行完了，
     //否则，表明还未完全执行完毕。
     //因此，如果第二个参数是DISPATCH_TIME_FOREVER,那么返回值恒为0.
     //而且，执行这个函数的当前线程会停止，在经过第二个参数指定的时间后，或者group中的执行全部完成后，才会恢复。
     if (result == 0) {
     NSLog(@"属于group的全部处理执行完成");
     }
     else {
     NSLog(@"属于group的全部处理还未执行完成");
     }
```

#### dispath_barrier_async
在访问数据库或者文件时，使用串行队列，可以避免数据竞争的问题，但是效率不高。其实写入处理确实不能与其他写入处理以及包含读取处理的其他处理并行执行，但是如果读取处理只是跟读取处理并行执行，那么多个并行执行，并不会产生数据竞争的问题。**也就是说，为了高效率的访问，读取处理追加到并行队列中，写入处理在没有读取处理执行的状态下，追加到一个串行队列中即可。（在写入结束之前，读取处理不可执行。）**
这个时候使用dispath_barrier_async函数是不错的选择。该函数与使用dispatch_queue_create生产成的concurrent Queue一起使用。

🌰

```
    dispatch_queue_t readQueue = dispatch_queue_create("readQueue", DISPATCH_QUEUE_CONCURRENT);
    void(^blockForRead0)() = ^{
        NSLog(@"block for reading 0");
    };
    void(^blockForRead1)() = ^{
        NSLog(@"block for reading 1");
    };
    void(^blockForRead2)() = ^{
        NSLog(@"block for reading 2");
    };
    void(^blockForRead3)() = ^{
        NSLog(@"block for reading 3");
    };
    void(^blockForRead4)() = ^{
        NSLog(@"block for reading 4");
    };
    void(^blockForRead5)() = ^{
        NSLog(@"block for reading 5");
    };
    void(^blockForRead6)() = ^{
        NSLog(@"block for reading 6");
    };
    void(^blockForRead7)() = ^{
        NSLog(@"block for reading 7");
    };
    
    void(^blockForwrite)() = ^{
        NSLog(@"block for writing~~~");
    };
    
    
	dispatch_async(readQueue, blockForRead0);
    dispatch_async(readQueue, blockForRead1);
    dispatch_async(readQueue, blockForRead2);
    dispatch_async(readQueue, blockForRead3);
    dispatch_async(readQueue, blockForRead4);
    dispatch_async(readQueue, blockForRead5);
    dispatch_async(readQueue, blockForRead6);
    dispatch_async(readQueue, blockForRead7);
    
```
现在我们要在blockForRead3和blockForRead4之间执行一个写入处理，并将写入的内容读取到blockForRead4以及之后的处理中.如果只像这样：

```
     dispatch_async(readQueue, blockForRead0);
     dispatch_async(readQueue, blockForRead1);
     dispatch_async(readQueue, blockForRead2);
     dispatch_async(readQueue, blockForRead3);
     
     
     dispatch_async(readQueue, blockForwrite);
     
     
     dispatch_async(readQueue, blockForRead4);
     dispatch_async(readQueue, blockForRead5);
     dispatch_async(readQueue, blockForRead6);
     dispatch_async(readQueue, blockForRead7);
```

单纯的在blockForRead3和blockForRead4之间插入一个执行一个写入处理，那么根据并行队列的性质，很可能在之后的处理中，读取的数据与期待的不符，并且，如果追加多个写入处理，则可能发生的问题更多。
    
因此我们要使用dispath_barrier_async函数。

🌰使用方法如下：

```
    dispatch_async(readQueue, blockForRead0);
    dispatch_async(readQueue, blockForRead1);
    dispatch_async(readQueue, blockForRead2);
    dispatch_async(readQueue, blockForRead3);
    
    
    dispatch_barrier_async(readQueue, blockForwrite);
    
    //dispatch_barrier_async会等待当前追加到并行队列上的执行全部处理结束之后，再将执行的处理（这里是blockForwrite）追加到该并行队列中，
    //并且在dispatch_barrier_async函数追加的处理执行完成之后，这个并行队列再继续恢复一般的动作，后续的处理又开始并行执行。
    
    dispatch_async(readQueue, blockForRead4);
    dispatch_async(readQueue, blockForRead5);
    dispatch_async(readQueue, blockForRead6);
    dispatch_async(readQueue, blockForRead7);
    //使用非常简单，用dispatch_barrier_async代替dispatch_async即可。
    
```

#### dispatch_semaphore_t

可以使用dispatch_semaphore_t来实现线程锁。

🌰

```
    NSMutableArray* array = [[NSMutableArray alloc] init];
    dispatch_semaphore_t _lock = dispatch_semaphore_create(1);
    for (NSInteger i = 0; i < 10; i ++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            dispatch_semaphore_wait(_lock, DISPATCH_TIME_FOREVER);
            [array addObject:[[NSObject alloc] init]];
            dispatch_semaphore_signal(_lock);
        });
    }
    
```    

***

### NSOperation对比GCD的优势

1. NSOperation可以取消。
2. NSOperation可以设置操作的依赖关系。
3. NSOperation可以使用KVO来观察操作的状态，比如isCancelled，isFinished。
4. NSOperation可以方便的重用。

但是，GCD比NSOperaiotn性能更好 ：）

***


### 参考

《Objective-C高级编程》

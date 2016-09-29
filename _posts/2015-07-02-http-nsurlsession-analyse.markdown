---
layout:     post
title:      "iOS网络请求大型攻略 -- HTTP协议、NSURLSession、AFNetworking"
subtitle:   "从HTTP原理到NSURLSession的原理和使用再到开源框架AFNetworking的源代码分析。"
date:       2015-07-02
author:     "waynezxcv"
header-img: "img/post.png"
tags:
    - iOS
    - HTTP
---

## 目录

1. [网络基础--TCP/IP协议](#tcp) 
2. [HTTP协议的原理](#http)
3. [NSURLSession原理与使用方法](#session)
4. [AFNetworking 3.0分析](#afnetworking)

***
 
 
## 网络基础--TCP/IP协议
 <p id = "tcp"></p>

### TCP/IP分层管理

TCP/IP协议族里面重要的一点就是分层。TCP/IP协议族按层次分别分为以下4层：应用层、传输层、网络层和数据链路层。分层的好处是，把各层的接口部分规划好后，每个层次内部就可以自有设计了，HTTP协议处于应用层。

利用TCP/IP协议进行网络通讯，会通过分层顺序呢与对方进行通信，发送端从应用层往下走，接收端从底层往应用层走。以HTTP举例，首先是发送端的客户端在应用层发出一个想看某个Web页面的请求，在传输层（TCP协议）会把从应用层收到的数据进行分割，并在各个报文加上标记符号和端口号后转发给网络层。在网络层（IP协议），增加作为通信目的的MAC地址后转发给数据链路层。接收端的服务器在数据链路层接收到数据，按顺序往上层发送，一直到应用层，才算真正接收到客户端发来的HTTP请求。发送端在层与层之间传输数据时，每经过一层，便会加上一个属于该层的首部信息。反之在接收端，在层与层之间传递时，每经过一层，会把对应的首部信息去掉。

#### IP协议

IP协议处于网络层。其作用是负责把数据传送给对方。而要保证数据传送到对方那里，则需要满足各类条件。其中两个重要条件就是IP地址和MAC地址。

* IP地址指明了节点被分配到的地址，IP地址可变。
* MAC地址是指网卡所属的固定地址，基本不可变。

#### TCP协议

TCP协议处于传输层。TCP协议将大块数据切割从报文段，并能够确认数据是否能够传输给对方。为了准确无误的将数据送达目标处，TCP协议采用了**三次握手策略**。
TCP协议将数据包送出去之后，并不会对传送后的情况置之不理，它一定会向对方确认传输是否成功。
发送端首先向接收端发送一个带有SYN（Synchronize）标志的数据包给对方，接收端接收到以后会回传一个带有SYN/ACK(Acknowledgement)标志的数据包以示确认。最后发送方再回传一个带有ACK标志的数据包代表握手结束。

#### DNS服务
DNS服务跟HTTP协议一样，处于应用层。DNS提供通过域名查找IP，或者通过IP反推域名的服务。

#### HTTP协议与这些协议的关系

* HTTP协议：负责生产针对目标Web服务器的HTTP请求报文（客户端）和接收对Web服务器请求的处理（服务端）。
* TCP协议：为了通信方便，把HTTP请求报文分割成报文段，并通过三次握手确保报文段传输的可靠性。
* IP协议：搜索对方的地址，一边中转一边传输。

***


## HTTP协议
 <p id = "http"></p>

### HTTP协议用于客户端和服务器之间的通信
HTTP协议和TCP/IP协议内的众多其他协议相同，用于客户端和服务端之间的通信。HTTP协议规定，请求从客户端发出，最后服务器响应该请求并响应。请求肯定是从客户端开始的，服务端在没收到请求时，是不会响应的。

#### 报文的构成
* 请求报文

一个HTTP请求报文举例：

```
POST /form/entry HTTP/1.1

Host:waynezxcv.com
Connection:keep-alive
Content-Type:text/html
Content-Length:16

name=waynezxcv&password=psd123456

```

起始的"POST"表示请求服务的方法，“form/entry”是请求的URI，“HTTP/1.1”表示HTTP协议的版本号。
中间的部分是请求头。
最后的“name=waynezxcv&password=psd123456”是请求体。

* 响应报文

一个HTTP响应报文举例

```
HTTP/1.1 200 OK

Date:Tue,02 Jul 2015 06:50:15 GMT
Content-Length:320
Content-Type:text/html

<html>...</html>

```

其实的“HTTP/1.1”表示HTTP协议版本号，“200”是状态码，最后的“OK”是状态码原因短语。后面的部分是响应头和响应体。

#### HTTP协议是不保存状态的协议

HTTP协议自身不对请求和响应之间的通讯状态做保存。HTTP/1.1虽然无状态，但是为了实现期望的保持状态功能，引入了cookie技术。

#### URI与URL
HTTP协议使用URI来定位互联网上的资源，使得在互联网上的任意资源都可以访问到。URI用字符串标示某一互联网资源，而URL标示资源的地点，URL是URI的子集。

#### HTTP请求方法

常用的HTTP请求方法有：

* GET:用来请求访问已被URI识别的资源。
* POST：用来传输请求体。
* PUT：用来传输文件
* DELETE:用来删除文件。
* HEAD:HEAD方法与GET方法一样，只是不返回报文的响应体。
* OPTIONS：查询URI支持的方法。

#### 持久连接和管线化

HTTP初始版本中，没进行一次HTTP通信就要断开一次TCP连接。（也就是说，每次HTTP请求，服务器跟客户端都要经过三次握手建立TCP连接，然后HTTP请求结束后，又断开）。这会增大通信量的开销。所以在HTTP/1.1中，使用了持久连接的方法，只要任意一段没有明确提出断开连接，则保存TCP连接状态。在HTTP/1.1中，所以的连接默认都是持久连接。
持久连接使得管线化发送成为可能，从前发送请求后，需要等待响应后才能发送下一个请求，管线化技术使得可以进行异步请求，即同时并行发送多个请求。

#### Cookie

之前说过，HTTP协议是不保存状态的协议。而Cookie技术可以通过在请求和响应报文中假如Cookie来控制客户端的状态。
客户端发送请求后，Cookie会在服务端的响应报文中加入一个叫做Set-Cookie的响应头字段，通知客户端保存Cookie。当下次客户端再次请求时，客户端会在请求头中自动假如Cookie字段发送出去。服务器端发现客户端发送过来的Cookie后，会对比服务器上的记录，得出究竟是从哪一个客户端发来的信息，从而得到之前的状态信息。

### HTTP报文

HTTP报文分为请求报文（客户端）和响应报文（服务器端）。
请求报文由请求行、请求头和请求体构成。

```
POST /form/entry HTTP/1.1

Host:waynezxcv.com
Connection:keep-alive
Content-Type:text/html
Content-Length:16

name=waynezxcv&password=psd123456

```

响应报文由状态行、响应头和响应体构成。


```
HTTP/1.1 200 OK

Date:Tue,02 Jul 2015 06:50:15 GMT
Content-Length:320
Content-Type:text/html

<html>...</html>

```
状态码的类别：

```
1XX 信息性状态码，表示请求正在处理
2XX 成功状态码，表示请求正常处理完毕
3XX 重定向状态码，表示需要附加操作以完成请求（比如URI表示的资源位置已经改变，需要重新定向）
4XX 客户端错误状态码，表示服务器无法处理请求
5XX 服务器端错误状态码，表示服务器处理请求出错

```

### HTTP报文首部（请求头和响应头）

HTTP首部字段是构成HTTP报文的要素之一。在客户端和服务器通信的过程当中，无论是请求还是响应都会使用首部字段，它起到**传递额外重要信息的作用**。

#### HTTP/1.1 常见首部字段

##### 请求首部（请求头）字段

* Accept 用户代理可处理的媒体类型 
* Accept-Charset 优先的字符集
* Accept-Encoding 优先的内容编码 
* Accept-Language 优先的自然语言
* Authorization Web认证信息
* Host 请求资源所在的服务器
* Range 实体的字节范围请求
* Use-Agent HTTP客户端程序的信息

##### 响应首部（响应头）字段

* Accept-Range 是否接收字节范围请求
* Age 推算资源创建时间消耗
* Etag 资源的匹配信息
* Location 令客户端重定向至指定URI
* Proxy-Authenticate 代理服务器对客户端的认证信息
* Sever HTTP服务器的安装信息

##### 实体首部字段

* Allow 资源可支持的HTTP方法
* Content-Encoding 实体主体的适用编码方式
* Content-Language 实体主题的自然语言
* Content-Length 实体主题的大小（字节）
* Content-Location 对应资源的URI
* Content-Range 实体主题的位置信息
* Content-Type 实体主题的媒体类型
* Expires 实体主题过期的日期时间
* Last-Modified 资源最后的修改日期时间

***

### HTTPS

#### HTTP的缺点

* 通信内容使用明文，内容可能被窃听
* 不验证通信方的身份，因此可能遭遇伪装
* 无法证明报文的完整性，所以可能已遭到篡改

##### HTTPS

** HTTP + SSL = HTTPS **

HTTPS在建立TCP通信之前，通过SSL建立安全通信线路。

HTTPS解决了HTTP的缺点

* 机密处理防止窃听。通信的加密：用SSL建立安全通信线路后，就可以在这条线路上进行HTTP通信了。
* 查明对方的证书。SSL使用证书的手段，可以查明通信双发的身份，证书由权威组织颁布。
* 防止内容被篡改。SSL提供摘要功能，来确保数据的完整性。

** HTTP + 加密 + 认证 + 完整性保护 = HTTPS **


## NSURLSession

 <p id = "session"></p>

NSURLSession是苹果在iOS7新加入的网络请求接口，用来取代NSURLConnection。NSURLSession包括与之前相同的组件，例如NSURLRequest, NSURLCache等。NSURLSession的不同之处在于，它把NSURLConnection替换为NSURLSession, NSURLSessionConfiguration，以及3个NSURLSessionTask的子类：NSURLSessionDataTask, NSURLSessionUploadTask, 和NSURLSessionDownloadTask。与NSURLConnection相比，NSURLSession最直接的改善就是提供了配置每个会话的缓存，协议，cookie和证书政策（credential policies），甚至跨应用程序共享它们的能力。这使得框架的网络基础架构和部分应用程序独立工作，而不会互相干扰。每一个NSURLSession对象都是根据一个NSURLSessionConfiguration初始化的，该NSURLSessionConfiguration指定了上面提到的政策，以及一系列为了提高移动设备性能而专门添加的新选项。

**在普通的应用场景下NSURLSession与NSURLConnection相比没有什么优势，但是在程序切换到后台之后Background的Session就显得更加灵活了。**

### NSURLSession当中的主要类介绍

#### NSURLSessionTask

NSURLSessionTask是一个抽象子类，它有三个具体的子类是可以直接使用的：

* NSURLSessionDataTask
* NSURLSessionUploadTask
* NSURLSessionDownloadTask

这三个类封装了现代应用程序的三个基本网络任务：获取数据，比如JSON或XML，以及上传下载文件。


当一个NSURLSessionDataTask完成时，它具有关联的数据，而一个NSURLSessionDownloadTask完成时，它具有一个已下载文件的临时文件路径。 NSURLSessionUploadTask 继承了 NSURLSessionDataTask，因为服务器响应一个上传请求时，往往伴随着相关联的数据。所有任务均可撤销，也可以暂停和恢复。当一个下载任务被取消时，它可以选择创建恢复数据，然后可以传递给下一次新创建的下载任务，以便继续之前的下载。

#### NSURLSessionConfiguration

NSURLSessionConfiguration用于指定NSURLSession的工作模式。

* 默认会话模式（default）：工作模式类似于原来的NSURLConnection，使用的是基于磁盘缓存的持久化策略，使用用户keychain中保存的证书进行认证授权。

* 及时会话模式（ephemeral）：该模式不使用磁盘保存任何数据，以及所有和会话相关的caches，证书，cookies等都被保存在RAM中，因此当程序使会话无效，这些缓存的数据就会被自动清空。

* 后台会话模式（background）：该模式在后台完成上传和下载，在创建Configuration对象的时候需要提供一个NSString类型的ID用于标识完成工作的后台会话。

#### NSURLSession

创建NSURLSession的三种方法。

```
/*
 * The shared session uses the currently set global NSURLCache,
 * NSHTTPCookieStorage and NSURLCredentialStorage objects.
 */
+ (NSURLSession *)sharedSession;

```

第一种方式是使用静态的sharedSession方法，该类使用共享的会话，该会话使用全局的Cache，Cookie和证书。


```
/*
 * Customization of NSURLSession occurs during creation of a new session.
 * If you only need to use the convenience routines with custom
 * configuration options it is not necessary to specify a delegate.
 * If you do specify a delegate, the delegate will be retained until after
 * the delegate has been sent the URLSession:didBecomeInvalidWithError: message.
 */
 + (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration;
 
 
 ```
 
 第二种方式是通过sessionWithConfiguration:方法创建对象，也就是创建对应配置的会话，与NSURLSessionConfiguration合作使用
 
 
 ```

+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration delegate:(nullable id <NSURLSessionDelegate>)delegate delegateQueue:(nullable NSOperationQueue *)queue;

```

第三种方式是通过sessionWithConfiguration:delegate:delegateQueue方法创建对象，二三两种方式可以创建一个新会话并定制其会话类型。该方式中指定了session的委托和委托所处的队列。当不再需要连接时，可以调用Session的invalidateAndCancel直接关闭，或者调用finishTasksAndInvalidate等待当前Task结束后关闭。这时Delegate会收到URLSession:didBecomeInvalidWithError:这个事件。Delegate收到这个事件之后会被解引用。

***

### NSURLSession的使用工作流

如果我们需要利用NSURLSession进行数据传输我们需要：

* 创建一个NSURLSessionConfiguration，用于第二步创建NSSession时设置工作模式和网络设置。
* 创建一个NSURLSession。系统提供了两个创建方法：

```
sessionWithConfiguration:
sessionWithConfiguration:delegate:delegateQueue:

```

第一个粒度较低就是根据刚才创建的Configuration创建一个Session，系统默认创建一个新的OperationQueue处理Session的消息。
 
第二个粒度比较高，可以设定回调的delegate（注意这个回调delegate会被强引用），并且可以设定delegate在哪个OperationQueue回调，如果我们将其设置为[NSOperationQueue mainQueue]就能在主线程进行回调非常的方便。

* 创建一个NSURLRequest调用刚才的NSURLSession对象提供的Task函数，创建一个NSURLSessionTask。

```
根据职能不同Task有三种子类：
NSURLSessionUploadTask：上传用的Task，传完以后不会再下载返回结果；
NSURLSessionDownloadTask：下载用的Task；
NSURLSessionDataTask：可以上传内容，上传完成后再进行下载。

```

* 当不再需要连接调用Session的invalidateAndCancel直接关闭，或者调用finishTasksAndInvalidate等待当前Task结束后关闭。这时Delegate会收到URLSession:didBecomeInvalidWithError:这个事件。Delegate收到这个事件之后会被释放。

* 如果是一个BackgroundSession，在Task执行的时候，用户切到后台，Session会和ApplicationDelegate做交互。当程序切到后台后，在BackgroundSession中的Task还会继续下载。

***


### NSURLSession使用示例

#### 发送一个GET请求

```

- (void)sendGetRequest {
    NSURLSessionConfiguration* configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession* session = [NSURLSession sessionWithConfiguration:configuration];
    NSMutableURLRequest* request = [[NSMutableURLRequest alloc] initWithURL:[NSURL URLWithString:@"http://www.waynezxcv.com"]];
   NSURLSessionDataTask* task = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        NSLog(@"%@",data);
        NSLog(@"%@",[NSThread currentThread]);
    }];
    [task resume];
}

```

这样的请求，系统将默认新建一个NSOperationQueue来进行回调。

#### 发送一个GET请求并指定回调的NSOperationQueue


```

- (void)sendGetRequestOnTheQueue:(NSOperationQueue *)queue {
    NSURLSessionConfiguration* configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession* session = [NSURLSession sessionWithConfiguration:configuration delegate:self delegateQueue:queue];
    NSMutableURLRequest* request = [[NSMutableURLRequest alloc] initWithURL:[NSURL URLWithString:@"http://www.waynezxcv.com"]];
    NSURLSessionDataTask* task = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        NSLog(@"%@",data);
        NSLog(@"%@",[NSThread currentThread]);
        NSLog(@"%@",[(NSHTTPURLResponse *)response allHeaderFields]);
    }];
    [task resume];
}


```

这样将指定一个NSOperationQueue来接收回调。


#### 发送一个POST请求

```

- (void)sendPostRequest {
    NSURLSessionConfiguration* configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession* session = [NSURLSession sessionWithConfiguration:configuration];
    NSMutableURLRequest* request = [[NSMutableURLRequest alloc] initWithURL:[NSURL URLWithString:@"http://www.waynezxcv.com"]];
    request.HTTPMethod = @"POST";//设置请求方法
    NSString* params =@"name=waynezxcv&loc=china";
    request.HTTPBody = [params dataUsingEncoding:NSUTF8StringEncoding];//设置请求体
    NSURLSessionDataTask* task = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
        NSLog(@"%@",data);
        NSLog(@"%@",[NSThread currentThread]);
        NSLog(@"%@",[(NSHTTPURLResponse *)response allHeaderFields]);
    }];
    [task resume];
}

```

#### 使用NSURLDownloadTask来下载文件

```

- (void)fileDonwload {
    NSURL* url = [NSURL URLWithString:@"http://img.club.pchome.net/kdsarticle/2013/11small/21/fd548da909d64a988da20fa0ec124ef3_1000x750.jpg"];
    NSURLSessionConfiguration* defaultConfigObject = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession* defaultSession = [NSURLSession sessionWithConfiguration:defaultConfigObject delegate:self delegateQueue: [NSOperationQueue mainQueue]];
    NSURLSessionDownloadTask* downloadTask =[ defaultSession downloadTaskWithURL:url
                                                                completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error) {
                                                  if(error == nil) {
                                                      NSLog(@"Temporary file =%@",location);//文件下载的临时路径
                                                      NSError* err = nil;
                                                      NSFileManager* fileManager = [NSFileManager defaultManager];
                                                      
                                                      NSString* docsDir = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) objectAtIndex:0];
                                                      NSURL* docsDirURL = [NSURL fileURLWithPath:[docsDir stringByAppendingPathComponent:@"out.zip"]];//文件将要转移到的目标路径
                                                      
                                                      if ([fileManager moveItemAtURL:location
                                                                               toURL:docsDirURL
                                                                               error: &err]) {
                                                          NSLog(@"File is saved to =%@",docsDir);
                                                      }
                                                      else {
                                                          NSLog(@"failed to move: %@",[err userInfo]);
                                                      }
                                                      
                                                  }
                                                  
                                              }];
    [downloadTask resume];
}

```

如果我们需要监控文件下载的进度，就需要实现下列的代理方法。但是需要注意，要实现代理方法，就不能使用Block回调的API了，不然并不会收到回调。


```

//下载文件，接收到数据并写入到临时路径
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask didWriteData:(int64_t)bytesWritten totalBytesWritten:(int64_t)totalBytesWritten totalBytesExpectedToWrite:(int64_t)totalBytesExpectedToWrite {
    //You can get progress here
    NSLog(@"接收了: %lld 字节 (总共下载了: %lld 字节)  文件的预计大小: %lld 字节.\n",
          bytesWritten, totalBytesWritten, totalBytesExpectedToWrite);
}

//下载成功回调
- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask didFinishDownloadingToURL:(NSURL *)location {
    NSLog(@"临时文件路径 :%@\n", location);
    NSError* err = nil;
    NSFileManager* fileManager = [NSFileManager defaultManager];
    NSString* docsDir = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) objectAtIndex:0];

    NSURL* docsDirURL = [NSURL fileURLWithPath:[docsDir stringByAppendingPathComponent:@"out1.zip"]];
    if ([fileManager moveItemAtURL:location
                             toURL:docsDirURL
                             error: &err]) {
        NSLog(@"文件已经存储到了路径 =%@",docsDir);
    } else {
        NSLog(@"文件存储到路径失败， %@",[err userInfo]);
    }

}

```

#### 使用NSURLDownloadTask来在后台下载文件

使用NSURLDownloadTask来在后台下载文件的工作流程如下。

1. 创建一个支持后台下载的NSURLSessionConfiguration，指定backgroundSessionConfiguration的Identifier。
2. 通过之前的NSURLSessionConfiguration，创建一个NSURLSession。
3. 启动NSURLSessionDownloadTask。
4. 在AppDelegate中实现``- (void)application:(UIApplication *)application handleEventsForBackgroundURLSession:(NSString *)identifier
  completionHandler:(void (^)())completionHandler;``代理方法。并将completionHandler保存起来传递给NSURLSession
  
4. 实现``-(void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session;``代理方法。


```

- (void)fileDownloadOnBackground {
    NSLog(@"file donwload");
    NSURL* url = [NSURL URLWithString:@"http://img.club.pchome.net/kdsarticle/2013/11small/21/fd548da909d64a988da20fa0ec124ef3_1000x750.jpg"];
    NSURLSessionConfiguration* backgroundConfigObject = [NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:@"com.waynezxcv.downloadtask1"];
    NSURLSession* backgroundSession = [NSURLSession sessionWithConfiguration:backgroundConfigObject delegate:self delegateQueue: [NSOperationQueue mainQueue]];
    NSURLSessionDownloadTask* downloadTask =[backgroundSession downloadTaskWithURL:url];
    [downloadTask resume];
    //在后台下载，不能使用Block的API来接收回调，要使用代理。
}

- (void)application:(UIApplication *)application handleEventsForBackgroundURLSession:(NSString *)identifier
  completionHandler:(void (^)())completionHandler {
    NSLog(@"handleEventsForBackgroundURLSession:%@",identifier);
    self.completionHandler = completionHandler;
}

// 后台session下载完成
-(void)URLSessionDidFinishEventsForBackgroundURLSession:(NSURLSession *)session {
    NSLog(@"后台下完完毕%@ .\n", session);
    AppDelegate * delegate =(AppDelegate *)[[UIApplication sharedApplication] delegate];
    if(delegate.completionHandler) {
        void (^handler)() = delegate.completionHandler;
        handler();
    }
}

    
```


##### 其他一些代理方法

###### 恢复下载


```
// 恢复下载，fileOffset是上次下载的偏移量。

- (void)URLSession:(NSURLSession *)session downloadTask:(NSURLSessionDownloadTask *)downloadTask didResumeAtOffset:(int64_t)fileOffset expectedTotalBytes:(int64_t)expectedTotalBytes {

}


```


###### 下载完成（不一定是成功，区别之前的代理方法）

下载任务手动取消，下载失败，下载完成都会收到回调。

可以通过error的useInfo拿到resumeData，并缓存起来。再次启动任务时，通过设置请求头的range来实现断点续传。``[request setValue:range forHTTPHeaderField:@"Range"];``


```

// 下载完成
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
     NSData *resumeData = nil;
    if (error) {
        resumeData = [error.userInfo objectForKey:NSURLSessionDownloadTaskResumeData];
    }
}


```


#### 使用NSURLUploadTask来实现文件上传

上传数据的时候，一般要使用RESTful API里的PUT或者POST方法。所以，要通过这个类来设置一些HTTP配置信息。设置HTTP请求头的方法：


```
addValue:forHTTPHeaderField:
setValue:forHTTPHeaderField:

```

##### 三种类型的数据的上传

* NSData 

如果对象已经在内存里使用以下两个函数初始化

```
uploadTaskWithRequest:fromData: 
uploadTaskWithRequest:fromData:completionHandler: 
```

* File

如果对象在磁盘上，这样做有助于降低内存使用。

```
 uploadTaskWithRequest:fromFile:
 uploadTaskWithRequest:fromFile:completionHandler:
 
 ```
 
 * Stream
 
 使用这个API来创建uploadtask
 
 ```
 uploadTaskWithStreamedRequest:

 ```

注意，这种情况下一定要提供Server需要的请求头，例如Content－Type和Content－Length。使用Stream一定要实现这个代理方法，因为Session没办法在重新尝试发送Stream的时候找到数据源。（例如需要授权信息的情况）。这个代理函数，提供了Stream的数据源。


```
URLSession:task:needNewBodyStream:

```


##### 监控上传进度

可以通过下面的代理方法来监控上传进度

```
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task
                                didSendBodyData:(int64_t)bytesSent
                                 totalBytesSent:(int64_t)totalBytesSent
                       totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend;
                      
```

#### 上传图片示例

```

- (void)imageUpload {
    
    NSMutableURLRequest* request = [NSMutableURLRequest requestWithURL:[NSURL URLWithString:@"http:www.freeimagehosting.net/upload.php"]];
    [request addValue:@"image/jpeg" forHTTPHeaderField:@"Content-Type"];//设置请求头字段，上传的文件类型
    [request addValue:@"text/html" forHTTPHeaderField:@"@Accept"];//设置请求头字段，通知服务器返回的优先数据类型
    [request setHTTPMethod:@"POST"];//设置请求方法
    [request setCachePolicy:NSURLRequestReloadIgnoringCacheData];//设置缓存策略
    [request setTimeoutInterval:20];//设置超时时间
    
    
    NSURLSessionConfiguration* defaultConfigObject = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession* defaultSession = [NSURLSession sessionWithConfiguration:defaultConfigObject delegate:self delegateQueue: [NSOperationQueue mainQueue]];
    NSData* imagedata = UIImageJPEGRepresentation([UIImage imageNamed:@"image"],1.0);//需要上传的图片
    NSURLSessionUploadTask* uploadTask =
    [defaultSession uploadTaskWithRequest:request fromData:imagedata];
    [uploadTask resume];
}

- (void)URLSession:(NSURLSession *)session
              task:(NSURLSessionTask *)task
   didSendBodyData:(int64_t)bytesSent
    totalBytesSent:(int64_t)totalBytesSent
totalBytesExpectedToSend:(int64_t)totalBytesExpectedToSend {
    NSLog(@"上传了数据 :%lld 字节 ，累计上传的字节数 ：%lld ，总共需要上传的字节数 :%lld",bytesSent,totalBytesSent,totalBytesExpectedToSend);
}

```

:)

*** 

## AFNetworking 3.0

 <p id = "afnetworking"></p>

AFNetworking是iOS开发中常用的一个开源网络请求框架。它主要包括三个部分，一是对NSURLSession的封装，使API调用更简单，二是提供了对请求和响应数据的序列化功能，最后还提供了网络状态监控的功能。
AFNetworking的使用方法十分简单，这里就不介绍了。下面主要分析一下AFNetworking当中主要类的作用。

### AFNetworking的核心AFURLSessionManager

AFURLSessionManager 是 AFHTTPSessionManager 的父类。**AFURLSessionManager是对NSURLSession的封装**。提供了简便的API方法，来帮我们快速进行HTTP请求。它主要负责下面几个任务:

1. 负责创建和管理NSURLSession
2. 管理NSURLSessionTask
3. 实现NSURLSessionDelegate等协议中的代理方法
4. 使用AFURLSessionManagerTaskDelegate管理请求进度
5. 引入AFSecurityPolicy保证请求的安全
6. 引入AFNetworkReachabilityManager监控网络状态。

### AFURLRequestSerializaiton和AFURLResponseSerialization
AFURLRequestSerializaiton和AFURLResponseSerialization分别提供了对HTTP请求和响应的序列化功能。前者的主要作用是修改请求（主要是 HTTP 请求）的头部，提供了一些语义明确的接口设置 HTTP 头部字段。后者是处理响应的模块，将请求返回的数据解析成对应的格式。

**AFURLResponseSerialization 负责对返回的数据进行序列化 。
AFURLRequestSerialization 负责生成 NSMutableURLRequest，为请求设置 HTTP 头部，管理发出的请求。**

### AFNetWrokReachabilityManager

AFNetworkReachabilityManager 是对 SystemConfiguration 模块的封装，苹果的文档中也有一个类似的项目 [Reachability](https://developer.apple.com/library/ios/samplecode/Reachability/) 这里对网络状态的监控跟苹果官方的实现几乎是完全相同的。

***


## 参考

《图解HTTP》-- 上野宣

 [Using NSURLSession](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/URLLoadingSystem/Articles/UsingNSURLSession.html#//apple_ref/doc/uid/TP40013509-SW1)

[AFNetworking](https://github.com/AFNetworking/AFNetworking)

*** 
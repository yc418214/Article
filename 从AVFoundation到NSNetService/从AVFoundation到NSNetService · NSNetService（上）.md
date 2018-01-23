# 从AVFoundation到NSNetService · NSNetService（上）

---

<p>

&emsp;之前已经把制作视频的过程介绍完了，也提了一种优化方式，接下来说的是我当时想到的另一种优化方式。

<p>

&emsp;附上之前的链接：

- [从AVFoundation到NSNetService · 开篇
](http://www.jianshu.com/p/51536a0df6cf)
- [从AVFoundation到NSNetService · AVFoundation](http://www.jianshu.com/p/c19af278b583)
- [从AVFoundation到NSNetService · Core Video](http://www.jianshu.com/p/998719aee8d9)


&emsp;我当时是用iPhone 5c做测试的，不仅内存有限，处理速度也不够满意，而OS X和iOS共用很多框架，除了界面用的框架有很大不一样，其他框架像AVFoundation、Core Video等，OS X也有，那能不能用MacBook来做这件事呢？

&emsp;然后我就新建了一个macOS项目，把代码复制过去，稍微修改了一下，测试的结果是，20秒内就生成一个视频了。接下来的问题就是，怎么把iPhone收集的朋友圈数据发送到MacBook。

&emsp;我一开始采用的是NSNetService+NSStream，连接速度很快，不过有时候发数据会堵塞，而且MacBook端收到`NSData`生成图片的时候，发现图片失真，后面就用NSNetService+CocoaAsyncSocket的方案。

<p>

&emsp;首先简单介绍一下`Bonjour`。`Bonjour`技术包括发布、发现和解析服务等，可以在Mac OS X系统和其他操作系统（比如iOS）之间建立网络连接。关于`Bonjour`的维基百科介绍在[这里](https://en.wikipedia.org/wiki/Bonjour_(software))，[Apple官方文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/NetServices/Introduction.html#//apple_ref/doc/uid/TP40002445-SW1)也有很详尽的介绍，不过我还没啃完，知识点很多，但是使用起来并不难。我们通过`NSNetService`来发布和发现网络服务，发现服务还不够，还需要通过`Socket`建立网络连接才能进行数据传输。`CocoaAsyncSocket`这个库对`Socket`做了很好的封装，使用起来很方便，文档也很清晰，所以整个过程就是NSNetService+CocoaAsyncSocket。

&emsp;`CocoaAsyncSocket`的[头文件](https://github.com/robbiehanson/CocoaAsyncSocket/wiki/Reference_GCDAsyncSocket)，[简单用法](https://github.com/robbiehanson/CocoaAsyncSocket/wiki/Intro_GCDAsyncSocket)，[注意事项](https://github.com/robbiehanson/CocoaAsyncSocket/wiki/CommonPitfalls)。首先把[CocoaAsyncSocket](https://github.com/robbiehanson/CocoaAsyncSocket)下载下来，它支持TCP和UDP，我们建立的是TCP连接（关于TCP和UDP篇幅有限就不讨论了），无论是发送端还是接收端，我们只需要用到`GCDAsyncSocket.h`和`GCDAsyncSocket.m`这两个文件。

&emsp;铺垫了这么多，来看代码。这一篇就先介绍iPhone发送端，首先要有一个`YCSenderSocketManager`，头文件也很简单：

````
@interface YCSenderSocketManager : NSObject

+ (instancetype)sharedManager;

- (void)sendDataWithData:(NSData *)data;

@end


````

**我们把需要发送的数据放到数组里，每次只向`Socket`写入一个`NSData`，`Socket`写完数据后再发送下一个。m文件的属性有：**

````

@interface YCSenderSocketManager ()

@property (strong, nonatomic) NSNetService *netService;

@property (strong, nonatomic) GCDAsyncSocket *senderSocket;

@property (strong, nonatomic) NSMutableArray<NSData *> *datasArray;

@property (assign, nonatomic, getter = isConnectedToServer) BOOL connectedToServer;

@end


````

**首先是要解析服务（Mac OS X端负责发布服务），我们在两边规定好`domain`、`type`和`name`后，发送端就可以省去发现服务这个过程了，关于这三个参数自行看[文档](https://developer.apple.com/reference/foundation/nsnetservice/1417615-initwithdomain?language=objc)吧。所以`sendDataWithData:`的实现如下：**

````
- (void)sendDataWithData:(NSData *)data {
  if (!data || data.length == 0) {
	return;
  }
  @synchronized (self.datasArray) {
    //外部调用不一定在主线程，记得加锁啦
    [self.datasArray addObject:data];
  }
  if (!self.isConnectedToServer) {
    if (self.netService) {
      return;
    }
    //NSNetService在发现端的实例化方式
    self.netService = [[NSNetService alloc] initWithDomain:@"local." type:@"_YC._tcp." name:@"Yuchuan"];
    self.netService.delegate = self;
    //对指定domain、type和name的NSNetService进行地址解析，设置5秒超时
    [self.netService resolveWithTimeout:5.0];
    return;
  }
  [self sendDataInQueue];
}

- (void) sendDataInQueue {
  ...
}

````

**我们需要实现NSNetServiceDelegate协议，在NSNetService地址解析成功后连接Socket：**

````
#pragma mark - NSNetServiceDelegate

- (void)netService:(NSNetService *)sender didNotResolve:(NSDictionary *)errorDictionary {
  //错误信息可查看NSNetServicesError枚举
  NSLog(@"DidNotResolve, error : %@",  errorDictionary);
}

- (void)netServiceDidResolveAddress:(NSNetService *)sender {
    NSError *error = NULL;
    //连接Socket
    if (![self.senderSocket connectToAddress:sender.addresses.firstObject error:&error]) {
        NSLog(@"connectToAddress fail, address : %@, error : %@", sender.addresses.firstObject, error);
    }
}
...

#pragma mark - getter

- (GCDAsyncSocket *)senderSocket {
  if (_senderSocket) {
    return _senderSocket;
  }
  //在主线程处理GCDAsyncSocket回调
  _senderSocket = [[GCDAsyncSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_main_queue()];
  return _senderSocket;
}

````

**接着是实现GCDAsyncSocketDelegate协议，连接失败后进行重连，连接成功则开始写数据：**

````
#pragma mark - GCDAsyncSocketDelegate

- (void)socket:(GCDAsyncSocket *)sock didConnectToHost:(NSString *)host port:(uint16_t)port {
  NSLog(@"didConnectToHost host : %@, port : %hu", host, port);
  self.connectedToServer = YES;
  //连接成功，开始写数据
  [self sendDataInQueue];
}

- (void)socketDidDisconnect:(GCDAsyncSocket *)sock withError:(nullable NSError *)err {
  NSLog(@"socketDidDisconnect error : %@", err);
  //进行重连，自己定义重连机制
  ...

  //重连失败
  self.connectedToServer = NO;
  self.netService = nil;
}

//完整地写完一个NSData后的回调
- (void)socket:(GCDAsyncSocket *)sock didWriteDataWithTag:(long)tag {
  @synchronized (self.datasArray) {
    if (self.datasArray.count == 0) {
      return;
    }
    //写下一个数据
    [self sendDataInQueue];
  }
}

````

&emsp;接下来是向GCDAsyncSocket写数据，这里要注意的是，**TCP是一个流，写入Socket的数据会被连接到一起，并分成一片一片地发送，分片的长度也不是每个`NSData`各自的长度，因此数据到了接收端那边后，需要接收端来对数据进行分割和组合**。有好多种进行数据分割的方法，只要发送端和接收端都规定好就可以。

&emsp;`GCDAsyncSocket`提供两类分段读取数据的函数,`readDataToData:`和`readDataToLength:`,分别对应下面两种思路：

- 第一种思路，发送端在每个数据单元后面加上结束符来标记，接收端通过读取结束符来区分不同数据单元。`GCDAsyncSocket`官方支持四种结束符字符串：'\r\n'、'\r'、'\n'、'\0'，对应的`NSData`分别是`[GCDAsyncSocket CRLFData]`、`[GCDAsyncSocket CRData]`、`[GCDAsyncSocket LFData]`、`[GCDAsyncSocket ZeroData]`，发送端在向`Socket`写数据前，给每个`NSData`加上规定好的结束符就可以了。比如：

````
//发送端
NSMutableData *dataToSend = [[NSMutableData alloc] initWithData:originalData];
[dataToSend appendData:[GCDAsyncSocket CRLFData]];
//写数据
[socket writeData:dataToSend withTimeout:-1 tag:1];

//接收端
[socket readDataToData:[GCDAsyncSocket CRLFData] withTimeout:-1 tag:1];

````

- 我采用的是[另一种思路](https://github.com/robbiehanson/CocoaAsyncSocket/issues/89)，跟HTTP协议类似，在每个`NSData`前加上一个头部信息，记录这个`NSData`的长度。具体如下：

````
- (void)sendDataInQueue {
  NSData *originalData;
  @synchronized (self.datasArray) {
    originalData = self.datasArray.firstObject;
    [self.datasArray removeObject:originalData];
  }
  if (!originalData) {
    return;
  }
  int dataLength = (int)originalData.length;
  //用4个字节长度的NSData记录发送数据的长度
  NSData *dataLengthData = [NSData dataWithBytes:&dataLength length:sizeof(int)];
        
  NSMutableData *dataToSend = [[NSMutableData alloc] initWithData:dataLengthData];
  [dataToSend appendData:originalData];
  //写数据
  [self.senderSocket writeData:dataToSend withTimeout:-1 tag:1];
}

````

<p>

**关于`GCDAsyncSocket`，官方资料介绍得很详细了，发送端就这么简单啦，下一篇写Mac OS X接收端。**









# 从AVFoundation到NSNetService · NSNetService（下）

---

<p>

&emsp;[上一篇](http://www.jianshu.com/p/2729aeb78b24)完成了`Socket`在iOS的发送端， 这一篇介绍`Socket`在Mac OS X的接收端。

<p>

&emsp;首先新建一个MacOS项目，然后把[CocoaAsyncSocket](https://github.com/robbiehanson/CocoaAsyncSocket)的`GCDAsyncSocket.h`和`GCDAsyncSocket.m`文件加入项目中。

&emsp;然后要有一个`YCReceiverSocketManager`类来管理网络服务和接收数据。接收数据后不保存，而是通知delegate进行处理，同时有开启和关闭功能，所以头文件如下：

````
@class YCReceiverSocketManager;

@protocol YCReceiverSocketManagerDelegate <NSObject>

- (void)receiverSocketManager:(YCReceiverSocketManager *)receiverSocketManager
               didReceiveData:(NSData *)data;

@end

@interface YCReceiverSocketManager : NSObject

@property (weak, nonatomic) id<YCReceiverSocketManagerDelegate> delegate;

//是否开启接收数据
@property (assign, nonatomic, getter = isRunning, readonly) BOOL running;

+ (instancetype)sharedManager;

- (void)openServer;

- (void)closeServer;

@end

````

**跟发送端一样，接收端也是NSNetService+GCDAsyncSocket，并且**接收端在和一个新的`Socket`建立连接后，并不会持有这个`Socket`，如果要保持连接则要把`Socket`保存起来**，所以m文件会有这些属性：**

````
@interface YCReceiverSocketManager ()

@property (strong, nonatomic) NSNetService *netService;

@property (strong, nonatomic) GCDAsyncSocket *receiverSocket;
//保存建立连接的socket
@property (strong, nonatomic) NSMutableArray<GCDAsyncSocket *> *acceptedSocketsArray;

@property (assign, nonatomic, getter = isRunning, readwrite) BOOL running;

@end

````

<p>

&emsp;首先`YCReceiverSocketManager`要发布一个服务，发现端可以通过`NSNetServiceBrowser`来发现服务，如果知道`domain`、`type`和`name`也可以直接生成`NSNetService`并进行地址解析（上一篇发送端就是这么做的）。关于`domain`、`type`和`name`看[官方文档](https://developer.apple.com/reference/foundation/nsnetservice/1417615-initwithdomain?language=objc)就够啦。

&emsp;我们实例化`NSNetService`来发布服务，**它有两个实例化方法，我们用它的指定初始化方法`initWithDomain:type:name:port:`生成一个用来被发布在特定端口的服务；而上一篇用到的另一个初始化方法`initWithDomain:type:name:`生成一个用来被解析的服务，它会调用指定初始化方法并传入一个不可用的端口（`port`），因此要注意发布服务和解析服务分别要用特定的初始化方法**。

&emsp;我们先调用`GCDAsyncSocket`的`acceptOnPort:error:`方法，让`socket`开始监听指定端口的连接，并给`port`传入0，这样系统会自动分配一个可用的端口给`GCDAsyncSocket`，这样的好处是我们不用自己指定一个端口，也可以避免端口被占用的可能。有了端口就可以实例化`NSNetService`了，所以`openServer`的实现如下：

````
- (void)openServer {
  NSError *error = NULL;
  if (self.receiverSocket.delegate != self) {
    self.receiverSocket.delegate = self;
  }
  if (![self.receiverSocket acceptOnPort:0 error:&error]) {
    NSLog(@"acceptOnPort error : %@", error);
    return;
  }
  UInt16 port = self.receiverSocket.localPort;
    
  //生成NSNetService
  self.netService = [[NSNetService alloc] initWithDomain:@"local." type:@"_YC._tcp." name:@"Yuchuan" port:port];
  self.netService.delegate = self;
  [self.netService publish];
}
...

#pragma mark - getter

- (GCDAsyncSocket *)receiverSocket {
  if (_receiverSocket) {
    return _receiverSocket;
  }
  //主线程处理socket回调
  _receiverSocket = [[GCDAsyncSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_main_queue()];
  return _receiverSocket;
}

````

**接着实现NSNetServiceDelegate协议：**

````
#pragma mark - NSNetServiceDelegate

- (void)netServiceDidPublish:(NSNetService *)netService {
    NSLog(@"Bonjour Service Published: domain(%@) type(%@) name(%@) port(%zd)",
          netService.domain, netService.type, netService.name, netService.port);
    self.running = YES;
}

- (void)netService:(NSNetService *)ns didNotPublish:(NSDictionary *)errorDictionary {
    NSLog(@"Bonjour Service Published failed, error : %@", errorDictionary);
    self.running = NO;
}

````

**然后实现GCDAysncSocketDelegate协议：**

````
#pragma mark - GCDAysncSocketDelegate

//接收到新的socket连接
- (void)socket:(GCDAsyncSocket *)sock didAcceptNewSocket:(GCDAsyncSocket *)newSocket {
  NSLog(@"Accepted new socket from %@ : %hu", newSocket.connectedHost, newSocket.connectedPort);
    
  if (![self.acceptedSocketsArray containsObject:newSocket]) {
    [self.acceptedSocketsArray addObject:newSocket];
  }
  //开始接收数据，上一篇说到我们用4个字节长度的NSData来记录发送数据的长度
  //所以连接新Socket后先读取4个字节
  [socket readDataToLength:sizeof(int) withTimeout:-1 tag:0];
}

//断开socket连接
- (void)socketDidDisconnect:(GCDAsyncSocket *)socket withError:(NSError *)error {
  NSLog(@"socketDidDisconnect error : %@", error);
  if ([self.acceptedSocketsArray containsObject:socket]) {
    [self.acceptedSocketsArray removeObject:  
  }
}

//接收到数据的回调
- (void)socket:(GCDAsyncSocket *)socket didReadData:(NSData *)data withTag:(long)tag {
  ...
}

````

&emsp;我们每调用`GCDAsyncSocket`的`readDataToData:`或`readDataToLength:`，从`socket`读取一定数据，即指定`NSData`（如：`[GCDAsyncSocket CRLFData]`）之前的或指定长度的数据后，都会收到`socket:didReadData:withTag:`回调，我们在回调里按照发送端规定好的分割方式，对TCP数据流进行数据分割和组合，这里写了一个`GCDAsyncSocket`的分类来处理：

````
#pragma mark - private methods

//接收完一个数据单元的处理
- (void)finishAcceptingDataWithReceivedData:(NSData *)receivedData {
  ...
}
...

#pragma mark - GCDAysncSocketDelegate

...
- (void)socket:(GCDAsyncSocket *)socket didReadData:(NSData *)data withTag:(long)tag {
  __weak YCReceiverSocketManager *weakManager = self;
  [socket yc_handleReadData:data completionBlock:^(BOOL hasMoreData, NSData *receivedData) {
    if (hasMoreData) {
      //一个数据单元还没读取完
      return;
    }
    //读取完一个完整的数据单元
    [weakManager finishAcceptingDataWithReceivedData:receivedData];
    //继续接收下一个data
    [socket readDataToLength:sizeof(int) withTimeout:-1 tag:0];
  }];
}

````

**GCDAsyncSocket的分类负责处理接收到的数据，首先是头文件：**

````
typedef void (^YCHandleReadDataBlock)(BOOL hasMoreData, NSData *receivedData);

@interface GCDAsyncSocket (YCAddition)

- (void)yc_handleReadData:(NSData *)data completionBlock:(YCHandleReadDataBlock) completionBlock;

@end

````

&emsp;**TCP接收端有一个接收缓存区，新数据会先来到这里，应用调用读操作，读取缓存区中的数据**。应用如果不调用读操作，数据就会一直在缓存区中，虽然操作系统会自动调整接收缓存区的大小，不过缓存区太大的话会占用内存，而且由于**TCP的流量控制机制**，也会影响发送端数据的发送，因此我们要及时读取接收缓存区的数据。[这里](http://www.cubrid.org/blog/dev-platform/understanding-tcp-ip-network-stack/)有篇文章介绍得挺清楚的，更多细节感兴趣的话可以看看。

&emsp;我们需要一个常量，表示应用每次从socket读取数据的长度。从`GCDAsyncSocket`的实现看，**`GCDAsyncSocket`有两种数据加密方式，分别是`SecureTransport`和`CFStream`**，`CFStream`方式的话，API没有告诉我们`socket`还有多少数据可以读取，`GCDAsyncSocket`的做法是每次从`socket`读取32 \* 1024长度的数据；而`SecureTransport`方式知道`socket`缓存区总共有多少数据，`GCDAsyncSocket`会**一次性读取**并保存在自己创建好的**预缓存**中，并且`SecureTransport`自己也有**数据缓存**，主要进行数据解密，`GCDAsyncSocket`的做法是32 \* 1024加上这两部分缓存的可用长度作为每次读取的长度。（对于32 * 1024这个值，`GCDAsyncSocket`的说法是“read as much data as we can”）

&emsp;来看GCDAsyncSocket分类的m文件。它需要保存两个数据，一是当前读取的数据单元的原本的长度，二是已经读取的数据单元的数据。

````
static NSInteger const kSocketDataReadLengthEachTime = 32768;

@interface GCDAsyncSocket ()
//一个数据单元可能会分多次读取，保存当前数据单元已经接收的数据
@property (strong, nonatomic) NSMutableData *yc_receivedData;
//保存当前接收的数据单元的原本的长度
@property (assign, nonatomic) int yc_expectedDataLength;

@end

@implementation GCDAsyncSocket (YCAddition)

#pragma mark - public methods

- (void)yc_handleReadData:(NSData *)data completionBlock:(CXHandleReadDataBlock)completionBlock {
  int expectedDataLength = self.yc_expectedDataLength;
  if (expectedDataLength == 0) {
    //先读取当前数据单元的期望长度并保存起来
    [data getBytes:&expectedDataLength length:sizeof(int)];
    self.yc_expectedDataLength = expectedDataLength;
    //开始读取数据单元的数据，每次最多32 * 1024KB
    [self readDataToLength:MIN(expectedDataLength, kSocketDataReadLengthEachTime) withTimeout:-1 tag:0];
    return;
  }
  //拼接数据单元的数据
  [self.yc_receivedData appendData:data];
    
  NSInteger leftDataLength = expectedDataLength - self.yc_receivedData.length;
  BOOL hasMoreData = (leftDataLength != 0);
  NSMutableData *receivedData = self.yc_receivedData;
    
  if (hasMoreData) {
    //还没读取完数据单元的数据
    [self readDataToLength:MIN(leftDataLength, kSocketDataReadLengthEachTime) withTimeout:-1 tag:0];
  } else {
    //重置
    self.yc_receivedData = [NSMutableData data];
    self.yc_expectedDataLength = 0;
  }
  if (completionBlock) {
    completionBlock(hasMoreData, [receivedData copy]);
  }
}

#pragma mark - getter

- (int)yc_expectedDataLength {
    return [objc_getAssociatedObject(self, @selector(yc_expectedDataLength)) intValue];
}
...

#pragma mark - setter

- (void)setYc_expectedDataLength:(int)yc_expectedDataLength {
    objc_setAssociatedObject(self, 
                             @selector(yc_expectedDataLength), 
                             @(yc_expectedDataLength), 
                             OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
...


````

&emsp;《从AVFoundation到NSNetService · NSNetService》系列到这里就结束啦，涉及的知识点包括AVFoundation、Core Video、NSNetService、GCDAsyncSocket等（本来还有Core Text）。[偶然的一个想法](http://www.jianshu.com/p/51536a0df6cf)推动我接触了很多新的东西，很有收获，只是篇幅有限，没细讲，只是介绍了大概的框架，不过好多东西后面还会继续深挖，就先这样吧。








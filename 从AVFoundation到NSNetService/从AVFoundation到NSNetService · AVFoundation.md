# 从AVFoundation到NSNetService · AVFoundation

---

<p>

&emsp;接上一篇[从AVFoundation到NSNetService · 开篇](https://github.com/yc418214/Article/blob/master/%E4%BB%8EAVFoundation%E5%88%B0NSNetService/%E4%BB%8EAVFoundation%E5%88%B0NSNetService%20%C2%B7%20%E5%BC%80%E7%AF%87.md)，接下来介绍YCMakeVideoManager的具体实现。YCMakeVideoManager用到了AVFoundation和Core Video两个框架，这一篇先写AVFoundation好了。

<p>

&emsp;首先制作视频过程中会占用不少内存，用Instruments测量发现少则几十M多则上百甚至200M，这个跟帧率和图片绘制的尺寸有关，因为后面会频繁创建bitmap context（**优化点1:可以考虑重用**），图片绘制到context后，数据会被渲染到内存上，所以图片绘制的尺寸越大，每张图片占用的内存就越大，帧率越多，生成bitmap context的数量就越多；但是绘制的尺寸越小（相对于视频尺寸，比如视频是720 \* 960的话，图片绘制尺寸是200 \* 200），会导致图片不清晰，帧率越少，会导致影片不连贯（图片不动除外）。

&emsp;所以这里要考虑几个问题：

- 选择合适的帧率，在保证肉眼可见的连贯性的基础下，帧率尽可能低
- 整个过程是很耗时的操作，必须丢到后台队列
- 从Instruments的测试结果看，内存释放的时机是在整个视频生成后，也就是说这个过程中内存是一直在上升的，有一种说法是只要图片数据还在被引用，内存就不会释放（**优化点2:试试autoreleasepool**）。所以最好控制一次只执行一个制作视频的任务，这里我是用信号量来控制的。

&emsp;所以YCMakeVideoManager的属性有下面几个：

````

@property (strong, nonatonmic) dispatch_queue_t semaphoreQueue;

@property (strong, nonatomic) dispatch_semaphore_t makeVideoSemaphore;

@property (strong, nonatomic) NSMutableDictionary *completionBlocksDictionary;

@property (strong, nonatomic) NSFileManager *fileManager;


````

**semaphoreQueue是来阻塞任务的串行队列，这个队列很重要。首先这个队列不是用来执行任务的队列，因为后面有的函数是异步函数，这样的话串行队列不能起到限制任务数量的作用；其次是不能直接在global queue调用`dispatch_semaphore_wait`，因为`dispatch_semaphore_wait`会堵塞当前队列，新的任务进来的时候，系统发现当前线程被堵塞，会不断创建新的线程，在信号量未大于0之前这些任务都不会被执行，这些线程就会带来没必要的开销。**

````
#define WEAKSELF    __weak YCMakeVideoManager *weakSelf = self;

#pragma mark - life cycle

- (instancetype)init {
  self = [super init];
  if (self) {
    //平时我是写在getter方法里，为了看起来更清晰，放在init方法里
    _semaphoreQueue = dispatch_queue_create("com.yc.makeVideo.semaphoreQueue", DISPATCH_QUEUE_SERIAL);
    _makeVideoSemaphore = dispatch_semaphore_create(1);
    _completionBlocksDictionary = [NSMutableDictionary dictionary];
    _fileManager = [NSFileManager defaultManager];
  }
  return self;
}

#pragma mark - public methods

//在public方法控制任务数量
- (void)makeVideoWithImagesArray:(NSArray<UIImage *> *)imagesArray videoSize:(CGSize)videoSize userName:(NSString *)userName completionBlock:(YCMakeVideoCompletionBlock)completionBlock {
  //这里要注意，如果videoSize的宽和高不是16的倍数，最后生成的视频会失真
  CGFloat videoFitWidth = (videoSize.width / 16) * 16;
  CGFloat videoFitHeight = (videoSize.height / 16) * 16;
  CGSize videoFitSize = CGSizeMake(videoFitWidth, videoFitHeight);

  WEAKSELF
  YCMakeVideoCompletionBlock block = ^(NSString *filePath) {
    //最后的回调里把信号量加1，以执行下一个任务
    dispatch_semaphore_signal(weakSelf.makeVideoSemaphore);
    if (completionBlock) {
      completionBlock(filePath);
    }
  };
  //保存处理回调
  self.completionBlocksDictionary[userName] = block;

  dispatch_async(self.semaphoreQueue,^{
    //信号量减1变为0，阻塞semaphoreQueue
    dispatch_semaphore_wait(self.makeVideoSemaphore, DISPATCH_TIME_FOREVER);
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
       [self makeVideoWithImagesArray:imagesArray videoSize: videoFitSize userName:userName];
    });
  });
}

#pragma mark - private methods

//在private方法执行主要逻辑
- (void)makeVideoWithImagesArray:(NSArray<UIImage *> *)imagesArray videoSize:(CGSize)videoSize userName:(NSString *)userName {
  ...
}

````

<p>

&emsp;接下来简单介绍一下在绘制过程中用到的几个类:

#### AVAssetWriter
`AVAssetWriter`用来把媒体数据写进一个新的文件，在初始化的时候指定文件导出路径和文件格式，这东西不能复用，只能写一个文件，所以就不用放到property了。

#### AVAssetWriterInput
`AVAssetWriter`负责输出数据，`AVAssetWriterInput`则负责接收数据。

#### AVAssetWriterInputPixelBufferAdaptor
图片数据会被保存在Core Video框架的`CVPixelBufferRef`（下一篇讲Core Video再介绍）类里，而`AVAssetWriterInputPixelBufferAdaptor`负责拼接`CVPixelBufferRef`数据，并且持有一个回收池，负责回收`CVPixelBufferRef`。

<p>

&emsp;所以它们的关系如图所示：

![关系图](http://upload-images.jianshu.io/upload_images/449722-6596d02f576d0066.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<p>

&emsp;再看一个叫`CMTime`的结构体，它可以用来表达时间间隔和某个时刻。我们用到它的两个值，`value`和`timescale`，这两个值的关系是value／timescale = seconds。举个例子：

````
//推荐用600作为timescale的值，因为它是一些视频帧率，比如24，30，60等的倍数
static NSInteger const kTimescale = 600;

//假设我们要表达1秒24帧，初始时刻（第一帧）为：
CMTime startTime = CMTimeMake(0, kTimescale);
//每一帧之间的时间间隔为（单位：秒）：
CGFloat frameInterval = 1 / (CGFloat)24；
//用CMTime表示为（由于value的值是long long类型，这就是为什么推荐用600作为timescale的原因）：
CMTime timeInterval = CMTimeMake(frameInterval * kTimescale, kTimescale);
//第二帧的时刻则可以表示为（CMTimeAdd就不用介绍了，函数名写的很清楚了，计算的时候注意换算成相同的timescale）：
CMTime nextTime = CMTimeAdd(startTime, timeInterval);

````

&emsp;我们都知道视频是一帧一帧的画面连起来的，而CMTime可以很好地表达某个时刻，当需要在比较细的粒度里操作视频画面的时候会很有用，比如在视频的某个时段或时刻加文字、马赛克之类。

&emsp;CMTime就介绍到这，接着是具体代码，一些写过一次就几乎不会变的东西，比如AVAssetWriter的实例化，我一般会单独抽出来放在后面：

````
static NSInteger const kTimescale = 600;

- (void)makeVideoWithImagesArray:(NSArray<UIImage *> *)imagesArray videoSize:(CGSize)videoSize userName:(NSString *)userName {
  AVAssetWriter *videoWriter = [self videoWriterWithUserName:userName];
  NSParameterAssert(videoWriter);

  AVAssetWriterInput *writerInput = [self videoWriterInputWithVideoSize:videoSize];
  NSParameterAssert(writerInput);

  NSParameterAssert([videoWriter canAddInput:writerInput]);
  [videoWriter addInput:writerInput];

  AVAssetWriterInputPixelBufferAdaptor *adaptor = [self adaptorWithWriterInput:writerInput];

  [videoWriter startWriting];
  [videoWriter startSessionAtSourceTime:kCMTimeZero];
  
  //图片之间的时间间隔
  NSInteger imageTimeInterval = [self imageTimeInterval];
  CMTime timeInterval = CMTimeMake(imageTimeInterval * kTimescale, kTimescale);
  //当前时刻
  CMTime currentTime = CMTimeMake(0, kTimescale);

  NSInteger imagesCount = imagesArray.count;
  NSInteger i = 0;
  while (i < imagesCount) {
    if (writerInput.isReadyForMoreMediaData) {
      [self processImageWithImage:imagesArray[i] 
                        videoSize:videoSize
                         userName:userName
                      writerInput:writerInput 
               pixelBufferAdaptor:adaptor
                      currentTime:currentTime];
      currentTime = CMTimeAdd(currentTime, timeInterval);
      i++;
    }
  }
  
  [writerInput markAsFinished];
  __weak YCMakeVideoManager *weakSelf = self;
  [videoWriter finishWritingWithCompletionHandler:^{
    if (videoWriter.status == AVAssetWriterStatusCompleted) {
      //make video successfully
      [weakSelf finishWritingVideoWithUserName:userName];
    }
  }];
  
  //为了效率和节省内存，CVPixelBufferRef是可复用的，这一点下一篇再讨论，这里需要清理回收池的内存
  CVPixelBufferPoolRelease(adaptor.pixelBufferPool);
}

//对每张图片的处理
- (void)processImageWithImage:(UIImage *)image 
                    videoSize:(CGSize)videoSize
                     userName:(NSString *)userName
                  writerInput:(AVAssetWriterInput *)writerInput
           pixelBufferAdaptor:(AVAssetWriterInputPixelBufferAdaptor *)adaptor
                  currentTime:(CMTime)currentTime {
  ...
}

//视频已导出文件
- (void)finishWritingVideoWithUserName:(NSString *)userName {
  ...
} 

...

- (AVAssetWriter *)videoWriterWithUserName:(NSString *)userName {
  NSString *filePath = [self filePathWithUserName:userName];
  NSURL *fileURL = [NSURL fileURLWithPath:filePath];
  NSError *error = NULL;
  AVAssetWriter *videoWriter = [[AVAssetWriter alloc] initWithURL:fileURL fileType:AVFileTypeMPEG4 error:&error];
  if (!videoWriter) {
    NSLog(@"initialization fail, error : %@", error);
  }
  return videoWriter;
}

- (AVAssetWriterInput *)videoWriterInputWithVideoSize:(CGSize)videoSize {
  NSDictionary *outputSettings = @{ AVVideoCodecKey : AVVideoCodecH264,
                                   AVVideoWidthKey : @(videoSize.width),
                                   AVVideoHeightKey : @(videoSize.height) };
  AVAssetWriterInput *writerInput = [AVAssetWriterInput assetWriterInputWithMediaType:AVMediaTypeVideo outputSetting:outputSettings];
  return writerInput;
}

- (AVAssetWriterInputPixelBufferAdaptor *)adaptorWithWriterInput:(AVAssetWriterInput *)writerInput {
  NSDictionary *pixelBufferAttributes = @{ (__bridge NSString *)kCVPixelBufferPixelFormatTypeKey : @(kCVPixelFormatType_32ARGB) };
  AVAssetWriterInputPixelBufferAdaptor *adaptor = 
  [AVAssetWriterInputPixelBufferAdaptor assetWriterInputPixelBufferAdaptorWithAssetWriterInput:writerInput 
                                                                   sourcePixelBufferAttributes:pixelBufferAttributes];
  return adaptor;
}

//导出的文件路径
- (NSString *)filePathWithUserName:(NSString *)userName {
  NSString *fileName = [NSString stringWithFormat:@"%@.mp4", userName];
  NSArray *pathsArray = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
  NSString *filePath = [pathsArray.firstObject stringByAppendingPathComponent:fileName];
  //AVAssetWriter不会覆盖同名文件，所以需要检查一下文件路径是否已经存在
  if ([self.fileManager fileExistsAtPath:filePath]) {
    [self.fileManager removeItemAtPath:filePath error:NULL];
  }
  return filePath;
}

//图片之间的时间间隔
- (NSInteger )imageTimeInterval {
  if (![self.delegate respondsToSelector:@selector(imageTimeIntervalForVideoManager:)]) {
    //默认时间间隔
    return 1;
  }
  return [self.delegate imageTimeIntervalForVideoManager:self];
}

````

&emsp;视频完成后还有利用AVFoundation添加音轨的部分，但是篇幅太长不利于阅读和吸收，所以留到下一篇好了。

&emsp;下一篇是对每张图片的具体处理以及视频导出文件后添加音轨，涉及到Core Video框架的知识。







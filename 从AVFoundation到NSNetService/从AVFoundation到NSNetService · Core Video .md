# 从AVFoundation到NSNetService · Core Video 

---

&emsp;上一篇把制作视频的外部框架搭建好了，接下来是每张图片的处理。

&emsp;附上之前的链接：

- [从AVFoundation到NSNetService · 开篇
](https://github.com/yc418214/Article/blob/master/%E4%BB%8EAVFoundation%E5%88%B0NSNetService/%E4%BB%8EAVFoundation%E5%88%B0NSNetService%20%C2%B7%20%E5%BC%80%E7%AF%87.md)
- [从AVFoundation到NSNetService · AVFoundation](http://www.jianshu.com/p/c19af278b583)

<p>

&emsp;我们要实现的效果是图片移动，原理其实很简单，就是一帧一帧地改变图片在视频里面的绘制位置，这在开篇已经写好计算逻辑。无论是移动、放大，还是图片的处理，比如加上高斯模糊效果，都是delegate做的事情，YCMakeVideoManager不关心外部的业务逻辑，它只需要delegate告诉它一张图片多少帧，图片之间的时间间隔是多少，然后在每一帧的时刻调用delegate的绘制方法就可以了。

&emsp;图片数据会被渲染到内存，保存在`CVPixelBufferRef`类里，这是Core Video框架的一个类，在绘制图片每一帧的时候都会把数据保存到`CVPixelBufferRef`里，因此不能频繁地创建它，会占用相当大的内存。`AVAssertWriterInputPixelBufferAdaptor`持有一个`CVPixelBufferPoolRef`，它可以创建新的`CVPixelBufferRef`用来保存图片数据，并在`AVAssertWriterInput`读取数据后对`CVPixelBufferRef`进行回收复用，大大减少内存的占用。


````
//对每张图片的处理
- (void)processImageWithImage:(UIImage *)image 
                    videoSize:(CGSize)videoSize
                     userName:(NSString *)userName
                  writerInput:(AVAssetWriterInput *)writerInput
           pixelBufferAdaptor:(AVAssetWriterInputPixelBufferAdaptor *)adaptor
                  currentTime:(CMTime)currentTime {
  NSInteger imageTimeInterval = [self imageTimeInterval];
  NSInteger imageFrameCount = [self imageFrameCount];
  //每一帧的时间间隔
  CMTime frameInterval = CMTimeMake(imageTimeInterval * kTimescale / imageFrameCount, kTimescale);
  //当前帧的时刻
  CMTime currentFrameTime = currentTime;
  //CVPixelBufferRef复用池
  CVPixelBufferPoolRef pixelBufferPoolRef = adaptor.pixelBufferPool;
  
  NSInteger i = 0;
  while (i < imageFrameCount) {
    if (writerInput.isReadyForMoreMediaData) {
      CVPixelBufferRef pixelBufferRef = [self pixelBufferWithImage:image
                                                      atFrameIndex:i
                                                         videoSize:videoSize
                                                          userName:userName
                                                   pixelBufferPool:pixelBufferPoolRef];
      //拼接CVPixelBufferRef
      [adaptor appendPixelBuffer:pixelBufferRef withPresentationTime:currentFrameTime];
      //防止内存泄漏
      CVPixelBufferRelease(pixelBufferRef);

      currentFrameTime = CMTimeAdd(currentFrameTime, frameInterval);
      i++;
    }
  }
}

//返回保存有图片数据的CVPixelBufferRef
- (CVPixelBufferRef)pixelBufferWithImage:(UIImage *)image 
                            atFrameIndex:(NSInteger)frameIndex
                               videoSize:(CGSize)videoSize 
                                userName:(NSString *)userName
                         pixelBufferPool:(CVPixelBufferPoolRef)pixelBufferPool {
  ...
}

- (NSInteger)imageFrameCount {
  if (![self.delegate respondsToSelector:@selector(imageFrameCountsForVideoManager)]) {
    return 1;
  }
  return [self.delegate imageFrameCountsForVideoManager:self];
}


````

**接着是把UIImage数据保存到CVPixelBufferRef里，有以下几个步骤：**

0. 创建CVPixelBufferRef，它会占用一个内存块
1. 在对CVPixelBufferRef的内存块写入数据前，要先锁住内存块
2. 获取CVPixelBufferRef内存块的基地址
3. 把内存块的基地址作为参数，创建CGContextRef（位图上下文），这样当图片绘制在CGContextRef后，位图数据就保存在CVPixelBufferRef的内存块中了
4. 最后释放CGContextRef，并解锁CVPixelBufferRef的内存块

````

- (CVPixelBufferRef)pixelBufferWithImage:(UIImage *)image 
                            atImageIndex:(NSInteger)imageIndex
                            atFrameIndex:(NSInteger)frameIndex
                               videoSize:(CGSize)videoSize 
                                userName:(NSString *)userName
                         pixelBufferPool:(CVPixelBufferPoolRef)pixelBufferPool {
  CVPixelBufferRef pixelBufferRef = NULL;
  //由CVPixelBufferPoolRef来负责创建pixelBufferRef
  CVReturn status = CVPixelBufferPoolCreatePixelBuffer(NULL, pixelBufferPool, &pixelBufferRef);
  NSParameterAssert(status == kCVReturnSuccess && pixelBufferRef);
  
  CVPixelBufferLockBaseAddress(pixelBufferRef, 0);
  
  void *pixelBufferData = CVPixelBufferGetBaseAddress(pixelBufferRef);
  NSParameterAssert(pixelBufferData != NULL);

  CGColorSpaceRef rgbColorSpaceRef = CGColorSpaceCreateDeviceRGB();
  CGContextRef contextRef = CGBitmapContextCreate(pixelBufferData, 
                                                  videoSize.width,
                                                  videoSize.height,
                                                  8,
                                                  4 * videoSize.width,
                                                  rgbColorSpaceRef,
                                                  kCGImageAlphaNoneSkipFirst);
  
  if ([self.delegate respondsToSelector:@selector(makeVideoManagerDrawImage:context:image:atFrameIndex:userName:)]) {
    //自定义绘制
    [self.delegate makeVideoManagerDrawImage:self
                                    context:contextRef
                                      image:image
                               atFrameIndex:i
                                   userName:userName];
  }
}
  
  if (contextRef) {
    CGContextRelease(contextRef);
  }
  CGColorSpaceRelease(rgbColorSpaceRef);

  CVPixelBufferUnlockBaseAddress(pixelBufferRef, 0);

  return pixelBufferRef;
}  

````

**到这里我们完成了图片到视频的转换，视频文件会被写到一开始指定的路径。接下来我们来给视频加上音轨，让它更生动。音频文件要事先保存在应用沙盒里，并由delegate把路径提供给YCMakeVideoManager。**

````
typedef void(^YCAddAudioCompletionBlock)(NSString *videoFilePath);

//视频已导出文件
- (void)finishWritingVideoWithUserName:(NSString *)userName {
  //视频路径看上一篇
  NSString *videoFilePath = [self filePathWithUserName:userName];

  if (![self.delegate respondsToSelector(audioFilePathForMakeVideoManager:)]) {
    [self saveToCameraRollWithFilePath:videoFilePath userName:userName];
    return;
  }
  NSString *audioFilePath = [self.delegate audioFilePathForMakeVideoManager:self];

  __weak YCMakeVideoManager *weakSelf = self;
  YCAddAudioCompletionBlock completionBlock = ^(NSString *newVideoFilePath){
    [weakSelf saveToCameraRollWithFilePath:newVideoFilePath userName:userName];
  };
  [self addAudioWithVideoFilePath:videoFilePath 
                    audioFilePath:audioFilePath 
                  completionBlock:completionBlock];
} 

//保存到系统相册
- (void)saveToCameraRollWithFilePath:(NSString *)filePath userName:(NSString *)userName {
  ...
}

//添加音频
- (void)addAudioWithVideoFilePath:(NSString *)videoFilePath audioFilePath:(NSString *)audioFilePath completionBlock:(YCAddAudioCompletionBlock)completionBlock {
  ...
}

````

<p>

简单介绍一下用到的几个类：

###### AVMutableComposition
&emsp;`AVMutableComposition`可以把多个媒体文件（音频、视频）拼接到一起，说人话就是视频剪辑和添加音频，音频剪辑等。它就像一个容器，存放视频轨道和音轨。
###### AVAssetTrack
&emsp;`AVAssetTrack`把媒体文件封装成轨道，轨道可以被剪辑、拼接等。
###### AVMutableCompositionTrack
&emsp;`AVMutableCompositionTrack`可以操作`AVAssetTrack`，比如截取某个时间段的`AVAssetTrack`，也可以拼接多个`AVAssetTrack`。前面说的`AVMutableComposition`就是`AVMutableCompositionTrack`的容器。
###### AVAssetExportSession
&emsp;`AVAssetExportSession`负责导出文件，可以指定导出的质量和文件类型，因此也可以用于视频压缩。

**合成过程如图：**

![合成过程](http://upload-images.jianshu.io/upload_images/449722-52722850dcd47e16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<p>

**来看代码：**

````
//添加音频
- (void)addAudioWithVideoFilePath:(NSString *)videoFilePath audioFilePath:(NSString *)audioFilePath completionBlock:(YCAddAudioCompletionBlock)completionBlock {
  NSData *audioData = [NSData dataWithContentsOfFile:audioFilePath];
  if (!audioData || audioData.length == 0) {
    if (completionBlock) {
      completionBlock(videoFilePath);
    }
    return;
  }
  AVMutableComposition *composition = [AVMutableComposition composition];

  AVURLAsset *videoAsset = [AVURLAsset URLAssetWithURL:[NSURL fileURLWithPath:videoPath] options:nil];
  AVURLAsset *audioAsset = [AVURLAsset URLAssetWithURL:[NSURL fileURLWithPath:audioFilePath] options:nil];
 //添加视频轨道
 [self addCompositionTrackWithURLAsset:videoAsset 
                             mediaType:AVMediaTypeVideo
                           composition:composition
                              duration:videoAsset.duration];
 //添加音轨
 [self addCompositionTrackWithURLAsset:audioAsset 
                             mediaType:AVMediaTypeAudio
                           composition:composition
                              duration:videoAsset.duration];
 
  //原视频文件的文件名，去掉.mp4后缀
  NSString *videoFileName = [videoFilePath substringWithRange:NSMakeRange(0, videoFilePath.length - 4)];
  //导出文件的路径
  NSString *exportFilePath = [NSString stringWithFormat:@"%@_withAudio.mp4", videoFileName];

  AVAssetExportSession *assetExportSession = [self exportSessionWithComposition:composition exportFilePath:exportFilePath];
  [assetExportSession exportAsynchronouslyWithCompletionHandler:^{
    if(!completionBlock) {
      return;
    }
    BOOL isSuccessful = (assetExportSession.status == AVAssetExportSessionStatusCompleted);
    NSFileManager *fileManager = [NSFileManager defaultManager];
    if (isSuccessful && [fileManager fileExistsAtPath:videoFilePath]) {
      //把原来的视频删掉
      [fileManager removeItemAtPath:videoFilePath error:NULL];
    }
    completionBlock(isSuccessful ? exportFilePath : videoFilePath);
   }];
}

- (AVAssetExportSession *)exportSessionWithComposition:(AVMutableComposition *)composition exportFilePath:(NSString *)exportFilePath {
  AVAssetExportSession *assetExportSession = [[AVAssetExportSession alloc] initWithAsset:composition presetName:AVAssetExportPresetHighestQuality];
  //导出文件格式
  assetExportSession.outputFileType = AVFileTypeMPEG4;
  //导出文件路径
  assetExportSession.outputURL = [NSURL fileURLWithPath:exportFilePath];
  return assetExportSession;
}

//添加轨道
- (void)addCompositionTrackWithURLAsset:(AVURLAsset *)URLAsset 
                              mediaType:(NSString *)mediaType
                            composition:(AVMutableComposition *)composition
                               duration:(CMTime)duration {
  //生成轨道
  AVAssetTrack *assetTrack = [URLAsset tracksWithMediaType:mediaType].firstObject;
  AVMutableCompositionTrack *compositionTrack = [composition addMutableTrackWithMediaType:mediaType preferredTrackID:kCMPersistentTrackID_Invalid];
  //添加轨道
  [compositionTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, duration)
                            ofTrack:assetTrack
                             atTime:kCMTimeZero 
                              error:NULL];
}

````


**然后把合成后的文件保存到系统相册：**

````
typedef void(^YCPhotoPerformChangesCompletionHandler)(BOOL isSuccessful, NSError *error);
//保存到系统相册
- (void)saveToCameraRollWithFilePath:(NSString *)filePath userName:(NSString *)userName {
  YCMakeVideoCompletionBlock makeVideoCompletionBlock = self.completionBlocksDictionary[userName];

  NSURL *fileURL = [NSURL fileURLWithPath:filePath];
  dispatch_block_t changesBlock =^{
    [PHAssetChangeRequest creationRequestForAssetFromVideoAtFileURL:fileURL];
  };
  YCPhotoPerformChangesCompletionHandler completionHandler = ^(BOOL isSuccessful, NSError *error) {
    if (makeVideoBlock) {
      makeVideoCompletionBlock(filePath);
      [self.completionBlocksDictionary removeObjectForKey:userName];
    }
    if (isSuccessful) {
      NSLog(@"Save video to Photo Library Successful");
      return;
    }
    NSLog(@"Save video to Photo Library error : %@", error);
  };
  [[PHPhotoLibrary sharedPhotoLibrary] performChanges:changesBlock
                                    completionHandler:completionHandler];
}

````

<p>

&emsp;整个过程到这里就结束了，但是从运行结果上看，我选了30张图片，每张图片24帧，生成一个视频差不多需要接近2分钟（iPhone 5c）。我想到两种优化方式：

- 首先是绘制每张图片的每一帧的过程是同步的，可以把图片分为几组，同时生成几个视频，最后再通过AVMutableComposition拼接起来。考虑到图片转换为CVPixelBufferRef的内存占用问题，图片的组数也需要控制，不能太多。不过这种优化方式我还没尝试，当时想到的是第二种优化方式。
- 第二种优化方式，下一篇再讨论。

# 从AVFoundation到NSNetService · 开篇

---

&emsp;iOS 10关于iMessage和照片的更新挺让人惊喜的，当时被照片的回忆功能触动到了。后来我就想到，把朋友圈的数据拉下来，做成一个2016回忆录。现在实现的效果是，把图片转成视频，并且图片可以移动或者放大，然后把朋友圈的配文、时间、地点都加上去。[示例视频下载链接点我](http://ohhjex6kk.bkt.clouddn.com/chuxin?attname=chuxin.mp4) 

&emsp;过程中遇到了挺多坑，但还是学到不少东西的，那就记录一下吧。

<p>

&emsp;首先新建一个核心类YCMakeVideoManager，负责接收图片，制作视频，加配乐，最后导出文件。为了让这个类专注于AVFoundation和Core Video框架，关于图片的具体绘制逻辑还是让delegate去处理，所以它的头文件会长这样：

````
@class YCMakeVideoManager;
@protocol YCMakeVideoDelegate<NSObject>

//图片的绘制逻辑
- (void)makeVideoManagerDrawImage:(YCMakeVideoManager *)makeVideoManager context:(CGContextRef)contextRef image:(UIImage *)image atFrameIndex:(NSInteger)frameIndex userName:(NSString *)userName;
//每张图片之间的时间间隔
- (NSInteger)imageTimeIntervalForVideoManager:(YCMakeVideoManager *)makeVideoManager;
//每张图片有多少帧
- (NSInteger)imageFrameCountsForVideoManager:(YCMakeVideoManager *)makeVideoManager;
//音频文件的路径
- (NSString *)audioFilePathForMakeVideoManager:(YCMakeVideoManager *)makeVideoManager;

@end

typedef void(^YCMakeVideoCompletionBlock)(NSString *filePath);

@interface YCMakeVideoManager : NSObject

@property (weak, nonatomic) id<YCMakeVideoDelegate> delegate;

+ (instancetype)sharedManager;

- (void)makeVideoWithImagesArray:(NSArray<UIImage *> *)imagesArray videoSize:(CGSize)videoSize userName:(NSString *)userName completionBlock:(YCMakeVideoCompletionBlock)completionBlock;

@end

````

**制作一个视频需要的数据有视频的尺寸、图片之间的时间间隔、每张图片的帧数（如果图片是静止不动的，则返回1就可以了）等，为了让这些数据便于修改，有一个YCMakeVideoUtil的类提供这些数据，如下：**

````
@interface YCMakeVideoUtil : NSObject

+ (CGSize)videoSize;
+ (CGFloat)imageTimeInterval;
+ (NSUInteger)imageFramesCount;

@end


````

**m文件直接返回静态常量就好了:**

````
static CGFloat const kVideoWidth = 720.f;
static CGFloat const kVideoHeight = 960.f;

static CGFloat const kImageTimeInterval = 2.f;

static NSUInteger const kImageFramesCount = 24;

@implementation YCMakeVideoUtil

+ (CGSize)videoSize {
 return CGSizeMake(kVideoWidth, kVideoHeight);
}
...


````

**关于YCMakeVideoManager的具体实现逻辑，篇幅有点长，留到下一篇再讨论好了，接下来是怎么使用YCMakeVideoManager，我们需要有一个业务逻辑类YCMakeVideoLogicManager，用来实现具体的绘制逻辑，m文件如下：**

````
#import “YCMakeVideoUtil.h”
#import “YCMakeVideoManager.h”

//我想实现的效果是，在竖直的视频里，横图可以移动
//图片移动的距离
static CGFloat const kImageMovingDistance = 40.f;

@interface YCImageDataManager () <CXVideoMakerDelegate>

@property (strong, nonatomic) YCMakeVideoManager *makeVideoManager;

@property (copy, nonatomic) NSArray<UIImage *> *imagesArray;

@property (assign, nonatomic) CGSize *videoSize;

@end

#pragma mark - private methods

- (instancetype)init {
  self = [super init];
  if (self) {
    _makeVideoManager = [YCMakeVideoManager sharedManager];
    _makeVideoManager.delegate = self;
    _videoSize = [YCMakeVideoUtil videoSize];
  }
  return self;
}

//开始制作视频
- (void)makeVideo {
  NSString *userName = ...;
  [self.makeVideoManager makeVideoWithImagesArray:self.imagesArray videoSize:self.videoSize userName:userName completionBlock:^(NSString *filePath) {
    NSLog(@“make video %@”, filePath ? @“successfully” : @“failed”);
  }];
}

#pragma mark - private methods

//返回每一帧图片在视频里面的绘制区域
- (CGRect)drawRectWithImage:(UIImage *)image atFrameIndex:(NSInteger)frameIndex {
  CGSize imageSize = image.size;
  if (imageSize.width < imageSize.height) {
    //由于我们只处理横图，不符合的图片就返回其他绘制尺寸，这里就不扩展了
    return …;
  }
  //这里就用视频的宽度作为图片的绘制高度好了，具体效果可以自己调节
  CGFloat imageDrawHeight = kVideoWidth;
  CGFloat imageDrawWidth = imageSize.width * imageDrawHeight / imageSize.height;
  CGFloat delta = imageDrawWidth - imageDrawHeight;
  //图片越接近正方形，移动的距离就会越短，所以对移动距离或图片绘制尺寸做一下调节
  CGFloat movingDistance = kImageMovingDistance;
  if (delta < movingDistance) {
    movingDistance = delta;
  } else {
    imageDrawWidth = videoWidth + movingDistance;
    imageDrawHeight = imageSize.height * imageDrawWidth / imageSize.width;
  }
  //第一帧图片的originX
  CGFloat firstImageFrameOriginX = -(imageDrawWidth - videoWidth + movingDistance) / 2;
  CGFloat frameProgress = frameIndex / (CGFloat)kImageFramesCount;

  CGFloat imageDrawOriginX = firstImageFrameOriginX + movingDistance * frameProgress;
  CGFloat imageDrawOriginY = (videoHeight - imageDrawHeight) / 2;
 
  return CGRectMake(imageDrawOriginX, imageDrawOriginY, imageDrawWidth, imageDrawHeight);
}

#pragma mark - YCMakeVideoDelegate

//绘制逻辑
-(void)makeVideoManagerDrawImage:(YCMakeVideoManager *)makeVideoManager context:(CGContextRef)contextRef image:(UIImage *)image atFrameIndex:(NSInteger)frameIndex userName:(NSString *)userName {
  //取上面计算好的图片区域，画上去就好了
  CGRect imageDrawRect = [self drawRectWithImage:image atFrameIndex:frameIndex];
  CGContextDrawImage(contextRef, imageDrawRect, image.CGImage);
}

- (NSInteger)imageTimeIntervalForVideoManager {
  return [YCMakeVideoUtil imageTimeInterval];
}

- (NSInteger)imageFrameCountsForVideoManager {
  return [YCMakeVideoUtil imageFramesCount];
}

 - (NSString *)audioFilePathForMakeVideoManager:(YCMakeVideoManager *)makeVideoManager {
  //mp3音频文件的路径
  return …;
}


````

<p>

**下一篇介绍YCMakeVideoManager的具体逻辑。**




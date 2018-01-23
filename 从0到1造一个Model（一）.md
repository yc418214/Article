
#从0到1造一个Model（一）

---

&emsp;狭义的model指的是针对具体业务封装的一个用于数据展示的对象，比如用一个带有消息id、消息发送者、接受者等属性的类，来表示一条消息；广义的model指的是model层，即除了视图和业务逻辑以外，跟数据有关的操作，比如数据的获取、加工、持久化等。[Mantle](https://github.com/Mantle/Mantle)、[YYModel](https://github.com/ibireme/YYModel)、[MJExtension](https://github.com/CoderMJLee/MJExtension)等第三方库面向的是狭义的model。

<p>

&emsp;对于赋值，Mantle和MJExtension采用的是KVC，YYModel则是直接调用getter、setter，关于KVC，[文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueCoding/Performance.html#//apple_ref/doc/uid/20002175-CJBDBHCB)是这么说的：
> Key-value coding is efficient, especially when you rely on the default implementation to do most of the work, but it does add a level of indirection that is slightly slower than direct method invocations. Use key-value coding only when you can benefit from the flexibility that it provides, or to allow your objects to participate in the Cocoa technologies that depend on it.

<p>

&emsp;相对直接调用getter、setter方法，KVC会稍微慢一点点，不过对于大多数应用来说，这点开销算不上瓶颈，而带来的好处是代码易写、可读性高，并且内部实现也包含了大部分类型转换；而对于直接调用getter、setter，则需要显性对数据进行类型转换，工作量大，好处是速度快，对于性能要求严格的可以采用这种方案。

<p>

&emsp;除了赋值，这些库还提供了其他功能，像自定义后台数据转换、数据归档等。不同项目的要求不一样，[这里](http://blog.ibireme.com/2015/10/23/ios_model_framework_benchmark/)有一篇对比的文章可以参考，选择适合自己项目的库，也可以根据需要定制自己的model。

<p>

&emsp;那就针对我在项目中的情况，简单地介绍造一个model的过程吧。

<p>

&emsp;**目前实现的功能点有：**

- 大部分model都用来封装服务器返回的数据，服务器用的key一般都很精简，而object-c的命名则是尽可能清晰，所以在赋值时需要自定义转换方式，提供key和属性的对应关系；
- 实现NSCoding以支持数据的持久化；而不一定所有的属性都需要持久化，因此提供一个接口，自选需要持久化的属性；
- 为了便于调试，也类似地提供接口，自选用于description的属性；
- 把model转换为服务器接受的数据格式，相当于第一点的逆过程

<p>

&emsp;我的目标是实现一个model基类，提供尽量全面的功能；项目中的业务model类继承以后，工作量尽可能少。

<p>

**首先定义一个协议来指定子类需要实现的一些方法：**

````
@protocol YCModelProtocol

@required
//后台key与属性的对应关系
+ (NSDictionary *)JSONKeysDictionary;
//用于NSCoding的属性
- (NSArray *)propertiesArrayForCoding;

@optional
//用于NSLog的属性
- (NSArray *)propertiesArrayForDescription;
//模仿Key-Value Validation，自定义逆转换过程
- (BOOL)transformForJSONWithValue:(inout id *)value forKey:(NSString *)key;

@end

````

**基类model头文件长这样：**

````
#import "YCModelProtocol.h"

@interface YCBaseModel : NSObject <YCModelProtocol, NSCoding>

//指定初始化方法，接收后台数据
+ (instancetype)modelWithJSONDictionary:(NSDictionary *)JSONDictionary;
//逆转换
- (NSDictionary *)JSONDictionary;
//所有属性，包括父类（如果父类也是model)
- (NSArray *)propertiesNameArray;

@end

````

**第一步是根据key和属性的对应关系，把后台数据转换成“属性-值”的形式，由于model的JSONKeysDictionary基本不会变，可以跟model类对象绑定并缓存起来，而不用每次都走消息转发；而转换过程丢到NSDictionary的分类去做：**

````
#pragma mark - life cycle

+ (instancetype)modelWithJSONDictionary:(NSDictionary *)JSONDictionary {
  //子类实例化的时候，[self class]输出的是子类的class，面试经常问的问题
  NSDictionary *JSONKeysDictionary = [self JSONKeysDictionaryWithModelClass:[self class]];
  //转换过程由分类去做，转换后字典的keys是model的属性
  NSDictionary *modelDictionary = [JSONDictionary yc_modelDictionaryWithJSONKeysDictionary:JSONKeysDictionary];
  return [[self alloc] initWithDictionary:modelDictionary];
}

- (instancetype)initWithDictionary:(NSDictionary *)modelDictionary {
  self = [super init];
  if (!self) {
    return nil;
  }
  if (![self configWithDictionary:[modelDictionary copy]]) {
    return nil;
  }
  return self;
}

#pragma mark - private methods

//赋值过程
- (BOOL)configWithDictionary:(NSDictionary *)dictionary {
  return ...;
}

//获取类的JSONKeysDictionary
+ (NSDictionary *)JSONKeysDictionaryWithModelClass:(Class)modelClass {
  return ...;
}

````

**由于Apple提供的isSubclassOfClass在两个class相等的时候也会返回YES，而我们在往上遍历父类的时候不需要考虑YCBaseModel的情况，所以先写一个NSObject分类的方法，方便后面的处理：**

````
@interface NSObject(Addition)

+ (BOOL)yc_isSubclassButNotEqualToClass:(Class)class {
    return [self isSubclassOfClass:class] && self != class;
}

@end


````

**关于获取JSONKeysDictionary的性能提升和线程安全：**

- 用CFMutableDictionaryRef缓存特定model的JSONKeysDictionary（也可以用关联对象，把JSONKeysDictionary绑定到类对象里）
- 用信号量来保护CFMutableDictionaryRef的存取

**所以JSONKeysDictionaryWithModelClass:的实现如下：**

````
+ (NSDictionary *)JSONKeysDictionaryWithModelClass:(Class)modelClass {
  //缓存
  static CFMutableDictionaryRef JSONKeysDictionaryCache;
  static dispatch_semaphore_t semaphore;
    
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    JSONKeysDictionaryCache = CFDictionaryCreateMutable(CFAllocatorGetDefault(),
                                                        0,
                                                        &kCFTypeDictionaryKeyCallBacks,
                                                        &kCFTypeDictionaryValueCallBacks);
    semaphore = dispatch_semaphore_create(1);
  });

  dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
  //类也是一个对象，用它的内存地址作为key
  NSDictionary *JSONKeysDictionary = CFDictionaryGetValue(JSONKeysDictionaryCache, (__bridge const void*)modelClass);
  dispatch_semaphore_signal(semaphore);
    
  if (!JSONKeysDictionary) {
    Class class = modelClass;
    //这里用NSMutableDictionary示范好了，追求性能可以用CFDictionaryCreateMutable
    NSMutableDictionary *dictionary = [NSMutableDictionary dictionary];
    while ([class cx_isSubclassButNotEqualToClass:[YCBaseModel class]]) {
      [dictionary addEntriesFromDictionary:[class JSONKeysDictionary]];
      //往上遍历父类，忽略YCBaseModel
      class = class_getSuperclass(class);
    }
    JSONKeysDictionary = [dictionary copy];
        
    if (JSONKeysDictionary) {
      dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
      CFDictionarySetValue(JSONKeysDictionaryCache,
                           (__bridge void*)modelClass,
                           (__bridge void*)JSONKeysDictionary);
      dispatch_semaphore_signal(semaphore);
    }
  }
  return JSONKeysDictionary;
}

````

**得到JSONKeysDictionary后，写一个NSDictionary分类的实例方法来处理，消息接收者是后台数据（即JSONDictionary）转换成modelDictionary：**

````
//遍历时传入的数据，用结构体保存参数对象地址，分别是用来取和存的两个字典
typedef struct {
  //后台数据
  void *JSONDictionary;
  //最终返回的数据
  void *modelDictionary;
} YCDictionaryFunctionContext;
//提供一个C函数，以供CFDictionaryRef遍历的时候调用，参数的格式是这样的:
//typedef void (*CFDictionaryApplierFunction)(const void *key, const void *value, void *context);
static void YCModelDictionaryConfiguration(const void *key, const void *value, void *context) {
  //这里的key是model的属性，value是对应的后台数据的key
  if (!context) {
    return;
  }
  //取出数据并进行类型转换
  YCDictionaryFunctionContext *functionContext = context;
  NSDictionary *JSONDictionary = (__bridge NSDictionary*)(functionContext->JSONDictionary);
  NSMutableDictionary *modelDictionary = (__bridge NSMutableDictionary*)(functionContext->modelDictionary);
  //兼顾keypaths的情况，像：@"user.id"
  NSArray *JSONKeysArray = [((__bridge NSString *)value) componentsSeparatedByString:@"."];
  id JSONValue = JSONDictionary;
  for (NSString *JSONKey in JSONKeysArray) {
    if (![JSONValue isKindOfClass:[NSDictionary class]]) {
      break;
    }
    JSONValue = JSONValue[JSONKey];
  }
  modelDictionary[(__bridge id)key] = JSONValue;
}

- (NSDictionary *)yc_modelDictionaryWithJSONKeysDictionary:(NSDictionary *)JSONKeysDictionary {
  NSMutableDictionary *modelDictionary = [NSMutableDictionary dictionary];
    
  YCDictionaryFunctionContext context;
  context.JSONDictionary = (__bridge void*)self;
  context.modelDictionary = (__bridge void*)modelDictionary;
  //遍历
  CFDictionaryApplyFunction((CFDictionaryRef)JSONKeysDictionary, YCModelDictionaryConfiguration, &context);
    
    return [modelDictionary copy];
}

````

<p>

**拿到modelDictionary就可以开始赋值了，这一篇就先到这吧。**







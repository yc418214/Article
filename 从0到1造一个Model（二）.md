#从0到1造一个Model（二）

---

&emsp;[接上一篇](http://www.jianshu.com/p/568725873f8f)。

<p>

**接着来看赋值过程。我采用的是Mantle的思路，用KVC + KVV来赋值和类型转换，如果有任意一个属性出错则model赋值失败，如下：**

````
//对value进行类型转换（如果需要的话）并赋值
static BOOL YCValidateAndSetValue(id object, NSString *key, id value) {
  ...
}

- (BOOL)configWithDictionary:(NSDictionary *)dictionary {
  if (!dictionary) {
    return NO;
  }
  if (![dictionary isKindOfClass:[NSDictionary class]]) {
    return NO;
  }
  __block BOOL isModelValid = YES;
  [dictionary enumerateKeysAndObjectsUsingBlock:^(NSString *key, id value, BOOL *stop) {
    id notNullValue = value;
    if ([notNullValue isEqual:[NSNull null]]) {
      notNullValue = nil;
    }
    BOOL isValid = YCValidateAndSetValue(self, key, notNullValue);
    if (!isValid) {  
      *stop = YES;
      isModelValid = NO;
    }
  }];
  return isModelValid;
}

````

<p>

###Key-Value Validation（KVV）

&emsp;KVV也是KVC的一部分，是指在赋值之前对值的自定义验证过程。举个例子，如果model有个`NSDate *timestamp`的属性，而服务器返回的数据是unix时间戳，如1485057199，则在赋值之前需要数据转换。KVC为KVV提供了[validateValue:forKey:error:](https://developer.apple.com/reference/objectivec/nsobject/1416754-validatevalue?language=objc)方法，[这里](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueCoding/Validation.html#//apple_ref/doc/uid/20002173-CJBDBHCB)有更详细的介绍。

<p>

**关于Key-Value Validation有几个要注意的点：**

- 首先KVC不会自动调用KVV，需要业务端自行调用；
- 只需要给对象发送`validateValue:forKey:error:`消息，内部实现会寻找符合`validate<Key>:error:`格式的方法并调用；当`key`为属性的名字时，我们就可以在KVC之前对某个属性进行类型转换；
- `validateValue:forKey:error:`传入的第一个参数为指向`*value`的指针，如果发现`*value`的值需要进行类型转换，我们应该重新生成一个对象并把`*value`指向新的对象，而不能直接修改`*value`原本指向的对象

**注意：**

&emsp;按照内存管理原则，谁生成谁释放，**如果从一个方法得到的对象不是用alloc/new/copy/mutableCopy生成的，则该对象会被自动注册到autoreleasepool再返回，换句话说，我们得到的是一个autoreleased的对象**。 

&emsp;一个方法的参数如果是一个**指向指针的指针**，那“被指向”的指针必须是**\__autoreleasing修饰**的。 比如，`NSError **error`，相当于`NSError * __autoreleasing *error`。因为方法内可能会生成一个新的对象并把“被指向”的指针指向它，但是这个指针不负责新的对象的释放，用\__autoreleasing修饰后，就可以在方法内把新对象注册到autoreleasepool再返回。

**YCValidateAndSetValue的具体实现如下：**

````
static BOOL YCValidateAndSetValue(id object, NSString *key, id value) {
  //指向指针的指针（比如id *、NSError **）默认是用__autoreleasing修饰的
  //即validateValue:forKey:error:第一个参数接受__autoreleasing修饰的参数
  //如果不写__autoreleasing，编译器也会默认生成一个用__autoreleasing修饰的临时对象作为参数
  __autoreleasing id validatedValue = value;
  @try {
    if (![object validateValue:&validatedValue forKey:key error:NULL]) {
      return NO;
    }
    [object setValue:validatedValue forKey:key];
    return YES;
  } @catch (NSException *exception) {
#if DEBUG
    @throw exception;
#else
    return NO;
#endif
  }
}

````

**所以上面关于时间戳的例子可以在子类model实现这样一个方法进行类型转换：**

````
- (BOOL)validateTimestamp:(inout *id)value error:(NSError *)error {
  if (!*value) {
    return YES;
  }
  if (![*value isKindOfClass:[NSString class]]) {
    return NO;
  }
  NSDate *timestamp = [NSDate dateWithTimeIntervalSince1970:((NSString *)*value).doubleValue];
  *value = timestamp;
  return YES;
}

````

<p>

**赋值过程差不多就这样，接下来看从model到JSONDictionary的逆过程。**

&emsp;我们遍历JSONKeysDictionary，用属性对应的后台key作为key，取出model中属性的值作为值；而类型转换的逆过程可以模仿KVV。

&emsp;（关于类型转换，Mantle采用的是[NSValueTransformer](https://developer.apple.com/reference/foundation/nsvaluetransformer)，这里有一篇[介绍](http://nshipster.cn/nsvaluetransformer/)。NSValueTransformer的好处是**如果封装得好**的话，转换过程写起来很简单，而且提供双向转换，坏处是学习成本高（相比KVV），而且需要对NSValueTransformer进行封装。而KVV的好处是学习成本低，可读性高，坏处是代码写起来相对冗余，而且没有逆向转换。不过逆过程可以**模仿KVV**的思路。）

**遍历JSONKeysDictionary:**

````
//额外数据
typedef struct {
  //model本身
  void *model;
  //最终结果
  void *JSONDictionary;
} YCJSONDictionaryTransformContext;

//转换为JSONDictionary
static void YCJSONDictionaryTransform (const void *key, const void *value, void *context) {
  if (!context) {
    return;
  }
  YCJSONDictionaryTransformContext *JSONDictionaryTransformContext = context;
  id model = (__bridge id)(JSONDictionaryTransformContext->model);
  NSMutableDictionary *JSONDictionary = (__bridge NSMutableDictionary *)(JSONDictionaryTransformContext->JSONDictionary);
    
  NSString *property = (__bridge NSString *)key;
  id JSONValue = [model valueForKey:property];
  if (!JSONValue) {
    return;
  }
  __autoreleasing id transformedJSONValue = JSONValue;
  //transformValue:forKey:在基类实现；同KVV，如果转换失败则返回NO
  if (![model transformValue:&transformedJSONValue forKey:property]) {
    return;
  }
  if (!transformedJSONValue) {
    return;
  }
  Class JSONValueClass = [transformedJSONValue class];
  //如果transformedJSONValue是一个Model
  if ([JSONValueClass cx_isSubclassButNotEqualToClass:[YCBaseModel class]]) {
    transformedJSONValue = [transformedJSONValue JSONDictionary];
  }
  JSONDictionary[(__bridge NSString *)value] = transformedJSONValue;
}

#pragma mark - public methods

//逆转换
- (NSDictionary *)JSONDictionary {
  Class modelClass = [self class];
  if (![modelClass conformsToProtocol:@protocol(YCModelProtocol)]) {
    return nil;
  }
  NSDictionary *JSONKeysDictionary = [modelClass JSONKeysDictionaryWithModelClass:modelClass];
  NSMutableDictionary *JSONDictionary = [NSMutableDictionary dictionary];
    
  YCJSONDictionaryTransformContext context;
  context.model = (__bridge void*)self;
  context.JSONDictionary = (__bridge void*)JSONDictionary;
  CFDictionaryApplyFunction((CFDictionaryRef)JSONKeysDictionary, YCJSONDictionaryTransform, &context);
    
  return [JSONDictionary copy];
}

#pragma mark - private methods

//检查子类是否实现了transformForJSONWithValue:forKey:或transformForJSONWith<Key>:方法，有则调用
- (BOOL)transformValue:(id *)value forKey:(NSString *)key {
  if ([self respondsToSelector:@selector(transformForJSONWithValue:forKey:)]) {
    return [self transformForJSONWithValue:value forKey:key];
  }
  //构造transformForJSONWith<Key>:方法SEL
  NSString *keyString = [key stringByReplacingCharactersInRange:NSMakeRange(0, 1)
                                                     withString:[[key substringToIndex:1] capitalizedString]];
  NSString *selectorString = [NSString stringWithFormat:@"transformForJSONWith%@:", keyString];
  if ([self respondsToSelector:NSSelectorFromString(selectorString)]) {
    return ((BOOL (*)(id, SEL, id*))objc_msgSend)(self, NSSelectorFromString(selectorString), value);
  }
  return YES;
}

````

<p>

**这一篇主要讲赋值过程和逆过程，下一篇再加上NSCoding和description。**




# 从0到1造一个Model（三）

---

&emsp;[接上一篇](https://github.com/yc418214/Article/blob/master/%E4%BB%8E0%E5%88%B01%E9%80%A0%E4%B8%80%E4%B8%AAModel%EF%BC%88%E4%BA%8C%EF%BC%89.md)。

<p>

&emsp;这一篇给model加上NSCoding和description。

<p>

### NSCoding

&emsp;为了让model支持本地持久化，我们实现NSCoding协议来序列化和反序列化model。我们在YCModelProtocol定义一个方法，**子类实现以返回需要进行编解码的属性数组**：

````
@protocol YCModelProtocol

@required
...
//用于NSCoding的属性
- (NSArray *)propertiesArrayForCoding;
...

@end

````

**model实现NSCoding协议:**

````
#pragma mark - NSCoding

- (instancetype)initWithCoder:(NSCoder *)aDecoder {
  self = [super init];
  if (self) {
    //由子类决定需要编解码的属性
    NSArray *propertiesArrayForCoding = [self propertiesArrayForCoding];
    for (NSString *property in propertiesArrayForCoding) {
      [self setValue:[aDecoder decodeObjectForKey:property] forKey:property];
    }
  }
  return self;
}

- (void)encodeWithCoder:(NSCoder *)aCoder {
  NSArray *propertiesArrayForCoding = [self propertiesArrayForCoding];
  for (NSString *property in propertiesArrayForCoding) {
    id value = [self valueForKey:property];
    [aCoder encodeObject:value forKey:property];
  }
}


````

**为了方便，我们在基类写一个获取所有属性的方法。我们利用关联对象，把子类的属性和类对象绑定起来，实现缓存：**

````
- (NSArray *)propertiesNameArray {
  Class modelClass = [self class];
  SEL associateKey = NSSelectorFromString(NSStringFromClass(modelClass));
  NSArray *propertiesNameArray = objc_getAssociatedObject(modelClass, associateKey);
  if (propertiesNameArray) {
    return propertiesNameArray;
  }
  NSMutableArray *propertiesArray = [NSMutableArray array];
  Class class = modelClass;
  //包括父类的属性（除了YCBaseModel）
  while ([class cx_isSubclassButNotEqualToClass:[YCBaseModel class]]) {
    unsigned int propertiesCount = 0;
    objc_property_t *properties = class_copyPropertyList(class, &propertiesCount);
    for (NSInteger i = 0; i < propertiesCount; i++) {
      NSString *propertyName = [NSString stringWithUTF8String:property_getName(properties[i])];
      [propertiesArray addObject:propertyName];
    }
    //记得释放
    free(properties);
    class = class_getSuperclass(class);
  }
  propertiesNameArray = [propertiesArray copy];
  objc_setAssociatedObject(modelClass, associateKey, propertiesNameArray, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
  return propertiesNameArray;
}

````

**子类实现propertiesArrayForCoding方法：**

````
- (NSArray *)propertiesArrayForCoding {
  //返回所有属性
  return [self propertiesNameArray];
  //or 返回自定义属性
//  return @[ NSStringFromSelector(@selector(...)),
			  ... ];
}

````

### Description

&emsp;同样在YCBaseModel定义一个方法，由子类决定用于NSLog的属性:

````
@protocol YCModelProtocol

@required
...
//用于description的属性
- (NSArray *)propertiesArrayForDescription;
...

@end

````

**重写description:**

````
#pragma mark - NSObject
//缩进的空格数量
static int i = 0;

- (NSString *)description {
  //子类实现propertiesArrayForDescription方法返回属性数组
  if (![self respondsToSelector:@selector(propertiesArrayForDescription)]) {
    return [super description];
  }
  i += 2;
  NSMutableString *descriptionString = [NSMutableString stringWithString:@"{ \n"];
  NSArray *propertiesForDescription = [self propertiesArrayForDescription];
  for (NSString *property in propertiesForDescription) {
    id value = [self valueForKey:property];
    //数组的处理
    if ([value isKindOfClass:[NSArray class]]) {
      //[NSString stringWithFormat:"%*s", i, " "] ，*s可以用来表示数量，这里表示有i个空格，用来缩进
      [descriptionString appendString:[NSString stringWithFormat:@"%*s%@ = ( \n", i, " ", property]];

      [(NSArray *)value enumerateObjectsUsingBlock:^(id valueItem, NSUInteger index, BOOL *stop) {
        [descriptionString appendString:[NSString stringWithFormat:@"%*s%@", i, " ", [valueItem description]]];
        if (index + 1 == ((NSArray *)value).count) {
          [descriptionString appendString:@" ) \n"];
        } else {
          [descriptionString appendString:@"\n"];
        }
      }];
    } else if ([value isKindOfClass:[NSDictionary class]]) {
      //字典的处理
      [descriptionString appendString:[NSString stringWithFormat:@"%*s%@ = { \n", i, " ", property]];
      i += 2;
      [(NSDictionary *)value enumerateKeysAndObjectsUsingBlock:^(id key, id value, BOOL *stop) {
        [descriptionString appendString:[NSString stringWithFormat:@"%*s%@ = %@ \n", i, " ", key, value]];
      }];
      i -= 2;
      [descriptionString appendString:[NSString stringWithFormat:@"%*s} \n", i, " "]];
    } else {
      [descriptionString appendString:[NSString stringWithFormat:@"%*s%@ = %@ \n", i, " ", property, value]];
    }
  }
  i -= 2;
  if (i != 0) {
    [descriptionString appendString:[NSString stringWithFormat:@"%*s}", i, " "]];
  } else {
    [descriptionString appendString:@"}"];
  }
  return [descriptionString copy];
}

````

**一次简单model的封装。**




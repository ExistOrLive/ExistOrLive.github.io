---
title:  "OC NSObject的-isEqual:和-hash 方法"
date:   2020-05-26
tags: [Objective-c]
excerpt_type: "html"
key: "ISEQUALANDHASHOFNSOBJECT"
article_header:
  type: cover
  image:
    src: /screenshot.jpg
---


## == 运算符

在 C 家族语言中，`==`运算符用于比较俩个变量的值是否相等。而OC中的对象都是使用指针变量来表示，因此当对于两个OC对象进行比较时，实际上比较的是两个指针的变量的值，即两个OC对象的地址。

```objc
int a = 0;
int b = 1;
int c = 0;

a == b // false

a == c // true 

NSString * d = @"Hello World";
NSString * e = @"Hello World";
NSString * f = [[NSString alloc] initWithFormat:@"Hello %@",@"World"];

d == e  // true 

e == f  //  false 

```

## NSObject 的 -isEqual: 方法 

而在OC的对象都实现了`NSObject`协议，协议中提供了

```objc

- (BOOL)isEqual:(id)object;
```

`-isEqual`是OC语言提供的用于判断两个OC对象是否相等的方法，而默认的实现就是返回`两个对象的地址是否一致。`

系统提供的以下类都重写了`isEqual`方法，实现了对象值的比较。

- NSAttributedString    `-isEqualToAttributedString:`
- NSData                `-isEqualToData:`  
- NSDate                `-isEqualToDate:`
- NSDictionary          `-isEqualToDictionary:`
- NSHashTable           `-isEqualToHashTable:`
- NSIndexSet            `-isEqualToIndexSet:`
- NSNumber              `-isEqualToNumber:`
- NSOrderedSet          `-isEqualToOrderedSet:`
- NSSet                 `-isEqualToSet:`
- NSString              `-isEqualToString:`
- NSTimeZone            `-isEqualToTimeZone:`
- NSValue               `-isEqualToValue:`


### 自定义isEqual

```objc
@interface Test : NSObject

@property(nonatomic,strong) NSString *keyProperty1;

@property(nonatomic,strong) NSString *keyProperty2;


@end

@implementation Test

- (BOOL) isEqual:(id) object {
  if(self == object) {
    return YES;                           //  比较对象地址是否相等
  }

  if(![object isKindOfClass:[self class]]){      // 比较是否是同一种类
    return NO;
  }

  return [self isEqualToTest:object];         // 对于关键属性进行比较

}

- (BOOL) isEqualToTest:(Test *) test {
  if([self.keyProperty1 isEqualToString:test.keyProperty1] && 
     [self.keyProperty2 isEqualToString:test.keyProperty2]){
       return YES;
     }
     return NO;
}

@end


```

### isEqual 的调用时机

除了主动直接调用`-isEqual:`方法之外，`NSArray`的`-containObject:`，`-indexOfObject:`在遍历数组时，会通过`-isEqual:`去判断对象是否相等


## NSObject 的 -hash 方法

```objc
- (NSUInteger) hash;
```

如果两个对象通过`-isEqual:`方法判断是相等的，那么这两个对象的`hash`也必须是相等的。非常重要的一点是： 如果某个类定义了`-isEqual:`方法并且想要在集合类中（`NSSet`，`NSMutableSet`等无序且不包含相同元素的容器）使用该类的对象，则必须在该类中重写`-hash`方法。

OC中的`NSSet`,`NSMutableSet`的实现就是hash表（无序且不包含相同元素的数据结构）


## hash表 

与数组和链表这种线性储存结构不同，哈希表会预先分配出一段内存，而元素在内存中的位置由元素的hash值决定，一个hash值对应一个位置。

当然也可能存在，两个不同的对象计算出的hash值是一样的，这种情况称为`哈希碰撞`。当发生`哈希碰撞`时，哈希表的内存段的同一个位置就会保存两个甚至更多的元素，这些元素会以链表的形式保存在这个位置。

在hash表中查找：

- 首先会根据查找对象的`-hash`方法计算出`hash`值，直接确定存储地址

- 如果该存储地址没有元素，则返回`false`。 

- 如果该存储地址存在一个或者多个元素，这些元素会以链表的形式保存。

- 遍历链表中的元素，通过元素的`-isEqual:`比较是否一致。

- 如果存在一致的元素，返回`true`，不存在返回`false`


一个有限存储空间的Hash表最理想的情况当然是每一个位置最多保存一个元素或者每个位置保存元素数量都大致相等，使得查找的时间复杂度最为接近O(1)。这样就对`-hash`方法有一定的要求，计算出的结果要尽量接近于`均匀分布`。

## 自定义-hash方法

`-hash`方法的默认实现是返回`对象的地址`

```objc
@implementation Test

//  对 keyProperty1 的 hash 值作位移，尽量避免hash碰撞 
- (NSUInteger) hash {
  return (self.keyProperty1.hash << 8) ^ (self.keyProperty2.hash);
}

@end

```


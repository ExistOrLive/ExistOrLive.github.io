---
title:  "OC NSObject的isEqual和hash"
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

## NSObject 的 isEqual 方法 

而在OC的绝大部分对象都实现了`NSObject`协议，协议中提供了

```objc

- (BOOL)isEqual:(id)object;
```

`isEqual`是OC语言提供的用于判断两个OC对象是否相等，而默认的实现就是返回`两个对象的地址是否一致。`

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





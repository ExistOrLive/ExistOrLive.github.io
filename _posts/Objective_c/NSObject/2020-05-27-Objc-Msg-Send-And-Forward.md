---
title:  "Objective-c 消息发送与消息转发"
date:   2020-05-27
tags: [Objective-c]
excerpt_type: "html"
key: "OBJCMSGSENDANDFORWARD"
article_header:
  type: cover
  image:
    src: /screenshot.jpg
---

在Objective-c语言中，调用某个对象的方法，称为向某个对象发送消息。不同于C语言的函数调用，OC的方法调用是动态的，在运行时才决定调用具体的方法实现。

 
```objc
int main(int argc, const char * argv[])
{
    @autoreleasepool {
        NSString * str = [[NSString alloc] init];
        NSLog(@"%@",str);
    }
    return 0;
}

```

将以上的OC代码通过`clang -rewrite-objc`转换为C/C++语言

```c

int main(int argc, const char * argv[])
{
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        NSString * str = ((NSString *(*)(id, SEL))(void *)objc_msgSend)((id)((NSString *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSString"), sel_registerName("alloc")), sel_registerName("init"));
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_32_bt767lwx1tvdwp4g5l1b3w2w0000gp_T_main_c3e70f_mi_0,str);
    }
    return 0;
}

```

可以看到OC代码中`+alloc`和`-init`的方法调用的部分，转变为`objc_msgSend()`的函数调用



## C语言的函数调用 

```c
#include <stdio.h>

void printHello(){
  printf("Hello World");
}

void printGoodBye(){
  printf("Good Bye");
}


void test(int a){
  if(a > 0) {
    printHello();
  } else {
    printGoodBye();
  }
}
```

C 语言在编译期就已经知道test()函数中会调用printHello()和printGoodBye()两个函数，会直接生成调用这些函数的指令，函数的入口地址会以硬编码的方法编译在指令中，这叫做`静态绑定`。


```c
#include <stdio.h>

void printHello(){
  printf("Hello World");
}

void printGoodBye(){
  printf("Good Bye");
}


void test1(int a){
  void(*func)();
  if(a > 0) {
    func = printHello;
  } else {
    func = printGoodBye;
  }
  func();
}
```
在编译期，test1()中会生成调用func函数的指令，但是func指向哪个具体的函数需要在运行期才能够确定，因此无法将函数入口地址硬编码到指令中。这就叫做`动态绑定`。

OC的方法方法调用就属于`动态绑定`，在编译期硬编码到指令中的是`objc_msgSend()`的函数调用，而调用的具体方法实现在运行期确定。

## 消息发送 objc_msgSend()

`objc_msgSend()` 函数实现的伪代码大致如下：先确定对象的类，再确定方法的具体实现并调用

```c
id objc_msgSend(id self, SEL _cmd, ...) {
  Class class = object_getClass(self);
  IMP imp = class_getMethodImplementation(class, _cmd);
  return imp ? imp(self, _cmd, ...) : 0;
}

IMP class_getMethodImplementation(Class cls, SEL sel)
{
    IMP imp;
    if (!cls  ||  !sel) return nil;
    imp = lookUpImpOrNil(cls, sel, nil, YES/*initialize*/, YES/*cache*/, YES/*resolver*/);
    // Translate forwarding function to C-callable external version
    if (!imp) {
        return _objc_msgForward;
    }
    return imp;
}

```

`class_getMethodImplementation` 的大致实现：

1. 首先会查询`fast map`快速映射表,如果找到则返回，未找到进入步骤2 (每一个类都有一块`fast map`的缓存，保存之前的匹配结果)

2. 查询接受者所属类的`方法列表`,如果找到则返回，未找到进入步骤3

3. 沿着`继承体系`继续向上查找，如果找到则返回，未找到进入`消息转发`的流程

## 消息转发 

当一个OC对象收到一条消息，OC对象会在所属类的`fast map`，`方法列表`以及在`继承树`中查询是否有对应的方法实现。如果能够找到，则直接调用方法实现；如果没有找到，则进入`消息转发`的流程。`消息转发`可以理解为在运行时发现消息没有对应的方法实现时，进行动态补救的过程。

### 动态方法解析（dynamic method resolution）

对象在收到无法解读的消息后，首先会进行`动态方法解析`，涉及以下两个方法：

```objc 

@interface NSObject<NSObject>

+ (BOOL)resolveClassMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

+ (BOOL)resolveInstanceMethod:(SEL)sel OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);

@end

```

`+resolveInstanceMethod:`在无法找到实例方法实现时调用；
`+resolveClassMethod:`在无法找到类方法实现时调用；

`在这两个方法中可以在运行时为当前类添加sel的方法实现，前提是方法实现的代码在编译时已经存在。`


```objc
//  例子

id dynamicGetter(id obj, SEL selector){
    return @"dynamicGetter";
}

void dynamicSetter(id obj, SEL selector,id value){
    NSLog(@"dynamicSetter %@",value);
}

@interface Test : NSObject

@property(nonatomic,strong) NSString * property1;

@end

@implementation Test

@dynamic property1;     // @dynamic 修饰属性，属性不生成对应的setter和getter方法

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    NSString * selStr = NSStringFromSelector(sel);
    if([selStr isEqualsToString:@"setProperty1:"]){
        class_addMethod(self, sel, (IMP)dynamicSetter, "v@:");
        return YES;
    } else if([selStr isEqualsToString:@"property1"]) {
        class_addMethod(self, sel, (IMP)dynamicGetter, "@@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];     // 子类无法处理时，记得调用父类的resolveInstanceMethod
}

@end


```

> 在以上代码中，`Test`类声明了`property1`属性，但是用`@dynamic`修饰了`property1`属性，`property1`不会生成对应的setter和getter方法。

> 当试图调用`property1`的setter或者getter方法，在`Test`类的方法列表和继承树中都找不到对应的方法实现，就进入`动态方法解析`，调用`+resolveInstanceMethod:`。在这个方法中为`property1`的setter或者getter方法添加方法实现，方法实现是已经存在的函数。

> `+resolveInstanceMethod:` 返回YES后，就重新走一遍`消息发送`的流程,此时`property1`的setter或者getter方法已经有了方法实现; 返回 NO 后，则继续`消息转发`的流程


### -forwardingTargetForSelector:

当`动态方法解析`无法处理未知消息，就会调用对象 `-forwardingTargetForSelector:`，试图将消息完整的转发给另一个对象，在方法中无法修改消息的内容(selector 和 参数)。

```objc
//  例子
@interface Test1 : NSObject

@property(nonatomic,strong) NSString * property1;

@end

@implementation Test1

@end



@interface Test : NSObject

@property(nonatomic,strong) NSString * property1;

@property(nonatomic,strong) Test1* test1;

@end

@implementation Test

@dynamic property1;     // @dynamic 修饰属性，属性不生成对应的setter和getter方法

- (id)forwardingTargetForSelector:(SEL)selector {
    if([selStr isEqualsToString:@"setProperty1:"] || 
       [selStr isEqualsToString:@"property1"]) {
         return _test1;   
    }

    return [super forwardingTargetForSelector:selector]; // 子类无法处理时，记得调用父类的forwardingTargetForSelector
               
}

@end

```

> 在以上代码中，当调用`Test`的`property1`的setter或者getter方法，找不到方法实现，动态方法解析没有处理时，调用到`-forwardingTargetForSelector:`方法，在这里将消息完整转发给`Test1`类的实例。

注意： `-forwardingTargetForSelector:`不能返回`self`,否则会陷入无限循环


### -forwardInvocation:

如果`-forwardingTargetForSelector:`不能够处理消息，就会调用到`-forwardInvocation:`

```objc
- (void)forwardInvocation:(NSInvocation *)anInvocation OBJC_SWIFT_UNAVAILABLE("");
```

如果转发算法到了这一步，会启动完整的消息转发机制。首先会创建`NSInvocation`对象，`NSInvocation`对象会将`receiver`，`selector`以及参数都封装起来，并作为参数传递到`-forwardInvocation:`方法。而在`-forwardInvocation:`方法中可以修改`receiver`，`selector`以及参数，使得`NSInvocation`成为一次有效的调用。

实现`-forwardInvocation:`方法时，若发现某调用操作不应由本类处理，则需要调用父类的同名方法。这样继承体系中每个类都有机会处理此调用请求，直到`NSObject`。在`NSObject`的`-forwardInvocation:`方法中，还会继而调用`doseNotRecognizerSelector:`抛出异常，表明消息最终未能处理。

### 消息转发的全流程


<div align="center"><img src="/public/Objective_c/NSObject/msg_forward1.png" alt="图1"  align="top" /></div>

在上图中， 接收者在每一步中均有机会处理消息。步骤越往后，处理消息的代价就越大。最好能够在第一步就处理完，这样运行期系统就可以将此方法缓存起来。如果此类的实例收到同名的消息，那么根本无需启动消息转发流程。若想在第三步将消息转给备用的接受者，那么建议还是在第二步进行转发，第三步的代价会比第二步大得多。



## 总结

- Objective-C的方法调用成为`消息发送`，消息由`receiver`，`selector`以及`参数`组成。

- Objective-C的消息发送是`动态绑定`的，在运行期确定方法实现。

- Objective-C的消息发送实际上调用的是`msg_send()`函数，在类的`fastmap`，`方法列表`以及继承体系中寻找方法实现。

- 如果某次方法调用无法找到方法实现，就进入`消息转发`的流程。

- 消息转发首先进行动态方法解析，调用`+resolveInstanceMethod:`或者`+resolveClassMethod:`，试图为消息添加对应的方法实现。

- `-forwardingTargetForSelector:` 会完整的转发消息，不会修改消息的`selector`和`参数`。

- 经过以上两步还不能处理消息，就调用`-forwardInvocation:`启动完整的消息转发机制，可以任意修改消息的`receiver`，`selector`以及`参数`。


## 参考文档

[Objective-C 消息发送与转发机制原理][1]

[Effective Objective-C 2.0][2]

这里贴上[大神](https://github.com/yulingtianxia)的消息发送和消息转发的流程图：

<div align="center"><img src="/public/Objective_c/NSObject/msg_forward2.jpg" alt="图1"  align="top" /></div>


[1]:http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/

[2]: https://baike.baidu.com/item/Effective%20Objective-C%202.0/22878393?fr=aladdin

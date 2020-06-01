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

## objc_msgSend()

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

3. 沿着继承体系继续向上查找，如果找到则返回，未找到进入`消息转发`的流程


## 消息转发 








## 参考文档

[Objective-C 消息发送与转发机制原理][1]


[1]:http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/
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


int main(int argc, const char * argv[]){
  printHello()
}


```


---
layout: post
title: "objective-c runtime 拾遗"
date: 2016-03-04 14:55:42 +0800
comments: true
categories: [iOS, runtime]
---

关于OC runtime、消息转发网上已经有很多文章，这里就不重复了，只是将一下不常见的、容易遗漏的列一下。

##动态添加类
我们通过`objc_allocateClassPair`、`class_addIvar`、`class_addMethod`、`objc_registerClassPair`来动态添加类。

`objc_registerClassPair` 其中除了设置类状态，做的最重要的事是生成`ivar_layout`(在支持GC的情况下，所以ios中是没有这一步的)，`ivar_layout`保存了类strong变量的内存视图，runtime依赖他来管理strong变量。

`ivar_layout`的结构[阳神的博客里](!http://blog.sunnyxx.com/2015/09/13/class-ivar-layout/)有描述，这里就不重复了。

`objc_registerClassPair`之后，类的`instanceSize`已经确定，这个新类已经可以投入使用，这时就不允许调用`class_addIvar`了。

`class_addIvar`主要是对`ivar_list`链表的操作，并相应的增加`instanceSize`。

参考：
[Apple 源码](http://www.opensource.apple.com/tarballs/objc4/)

##动态方法解析
一个OC方法的实现本质上就是一个简单的c函数，这个c函数至少要有self和_cmd两个参数。比如下面这个c函数：
```
void dynamicMethodIMP(id self, SEL _cmd) {
    // implementation ....
}
```
可以用`class_addMethod`来添加到现有的类中：
```
@implementation MyClass
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if (aSEL == @selector(resolveThisMethodDynamically)) {
          class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
          return YES;
    }
    return [super resolveInstanceMethod:aSEL];
}
@end
```
`resolveInstanceMethod`和`resolveClassMethod`可以用来动态提供一个方法实现。

OC runtime做消息转发（Message Forwarding）时，在调用`respondsToSelector:`和`instancesRespondToSelector:`之前，会先调用上面两个方法，让你有机会来动态添加方法实现。

OC支持一种动态属性允许你动态提供它的实现方法。
```
@dynamic propertyName;
```

[参考](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtHowMessagingWorks.html)
<!-- more -->

##消息转发
系统发送消息给一个对象时，如果这个对象不能响应这个消息，在报错之前，系统会调用这个对象的`forwardInvocation:` 让它有机会来处理这个消息。
顾名思义就是消息转发，通过将消息转发给另一个对象，使得一个对象可以模拟另外一个对象的能力：
```
/*
forwardInvocation 需要MethodSignature来创建NSInvocation对象，因此这个方法也要提供。
*/
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector  
{  
    NSMethodSignature* signature = [super methodSignatureForSelector:aSelector];  
    if (signature==nil) {  
        signature = [someObj methodSignatureForSelector:aSelector];  
    }  
    NSUInteger argCount = [signature numberOfArguments];  
    for (NSInteger i=0 ; i<argCount ; i++) {  
    }  
      
    return signature;  
}  

- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    if ([someOtherObject respondsToSelector:
            [anInvocation selector]])
        [anInvocation invokeWithTarget:someOtherObject];
    else
        [super forwardInvocation:anInvocation];
}
```
你可以用这种机制来模拟多继承。

要真实的模拟多继承，你还必须重写这些方法：`respondsToSelector:` 、`isKindOfClass:`、`instancesRespondToSelector:`、如果继承对象中有协议，还需重写`conformsToProtocol:`

##类型编码
是OC runtime所依赖的重要类型信息，编译器将参数及返回类型编码成字符串，并与方法selector关联。

@encode是个编译器操作符，用来获取一个类型的编码。所有可以用sizeof操作符的类型，都可以用@encode来获取它的类型。

常见编码见[官方文档](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)

---
layout: post
title: "class-swizzling, isa-swizzling and KVO"
date: 2016-03-07 20:44:20 +0800
comments: true
categories: [iOS, runtime, KVO]
---

看`ReactiveCocoa`源码的时候，被`RACSwizzleClass`卡住了，做了以下研究，并把注释后的代码放在文章最后，如果不想看过程可以直接跳到最后。

`RACSwizzleClass`中通过判断`[obj class] `和 `object_getClass(obj)`是否相同来执行不同的逻辑。
然而`[obj class]`和`object_getClass(obj)`有什么区别？他们不同到底意味着什么？

为此查阅了objc runtime的源代码，并整理了相关代码：[get class 相关代码](https://gist.github.com/agdsdl/a22666c8f64fed0dbbf5)

结论:

- 对于一个普通的obj（不是class），`[obj class] `和 `object_getClass(obj)`会返回一样的结果，就是该对象所属的类。
- 对于一个class，`[aclass class]`还是会返回该class，而`object_getClass(aclass)`会溯本归源返回aclass的`isa`，一般是返回该类的`meta class`

对象的`isa`链会一直指向哪里?见下图:

![objc 关系图](/images/class-diagram.jpg)

(图片来自http://blog.devtang.com/2013/10/15/objective-c-object-model/)

既然对于一个普通的obj（不是class），`[obj class] `和 `object_getClass(obj)`会返回一样的结果，那么`RACSwizzleClass`为什么要做相等性判断？

在苹果的文档中稍稍提到了一些：
[Key-Value Observing Implementation Details](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/KeyValueObserving/Articles/KVOImplementation.html)

简单的说，系统的KVO，是用`isa-swizzling`实现的。

`isa`指针，指向对象所属的类，类里面存储的是方法列表及其他一些信息。

当你给一个对象添加了observer之后，系统会修改该对象的`isa`指针，使其指向一个中间类（中间类重写了setting方法以实现KVO），这时的`isa`，就不是指向对象实际的类了。

所以，你应该用`class`(系统同时重写了class方法使其返回对象原来的类)方法，而不是`isa`(object_getClass)来取得对象所属的类。

<!-- more -->
写了个小小的测试程序来验证：
```
- (void)test{
    //Now create an instance 'myobj' of class 'MyObject'
    NSLog(@"Now create an instance 'myobj' of class 'MyObject'");
    MyObject *myobj = [[MyObject alloc] init];
    Class aclass = [myobj class];
    NSLog(@"[myobj class] returns:%@(%p)", aclass, aclass);
    Class aclass2 = object_getClass(myobj);
    NSLog(@"object_getClass(myobj) returns:%@(%p)", aclass2, aclass2);
    Class aclass3 = object_getClass([myobj class]);
    NSLog(@"object_getClass([myobj class]) returns:%@(%p)", aclass3, aclass3);

    //Now recursively call 'class'
    NSLog(@"Now recursively call 'class'");
    id obj = myobj;
    for (int i=0; i<4; i++) {
        Class aclass = [obj class];
        obj = aclass;
        NSLog(@"pass%d [obj class] returns %@(%p)", i+1, aclass, aclass);
    }

    // Now recursively call 'object_getClass'.
    NSLog(@"Now recursively call 'object_getClass'");
    obj = myobj;
    for (int i=0; i<4; i++) {
        Class aclass = object_getClass(obj);
        obj = aclass;
        NSLog(@"pass%d object_getClass(obj) returns %@(%p)", i+1, aclass, aclass);
    }

    
    // Now add KVO to myobj.
    NSLog(@"Now add KVO to myobj.");
    [myobj addObserver:self forKeyPath:@"title" options:NSKeyValueObservingOptionNew context:nil];
    NSLog(@"After KVO. [myobj class] returns %@(%p)", [myobj class], [myobj class]);
    NSLog(@"After KVO. object_getClass(myobj) returns %@(%p)", object_getClass(myobj), object_getClass(myobj));

    myobj.title = @"Hello, this is new title!";
    
    // Now remove KVO of myobj.
    NSLog(@"Now remove KVO of myobj.");
    [myobj removeObserver:self forKeyPath:@"title"];
    NSLog(@"KVO removed. [myobj class] returns %@(%p)", [myobj class], [myobj class]);
    NSLog(@"KVO removed. object_getClass(myobj) returns %@(%p)", object_getClass(myobj), object_getClass(myobj));
}
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context{
    if ([keyPath isEqualToString:@"title"]) {
        NSLog(@"KVO title changed:%@", [change objectForKey:NSKeyValueChangeNewKey]);
    }
}
```
运行结果：
```
2016-03-07 17:19:09.702 testOC[3925:131380] Now create an instance 'myobj' of class 'MyObject'
2016-03-07 17:19:09.704 testOC[3925:131380] [myobj class] returns:MyObject(0x106520120)
2016-03-07 17:19:09.704 testOC[3925:131380] object_getClass(myobj) returns:MyObject(0x106520120)
2016-03-07 17:19:09.704 testOC[3925:131380] object_getClass([myobj class]) returns:MyObject(0x1065200f8)
2016-03-07 17:19:09.705 testOC[3925:131380] Now recursively call 'class'
2016-03-07 17:19:09.705 testOC[3925:131380] pass1 [obj class] returns MyObject(0x106520120)
2016-03-07 17:19:09.705 testOC[3925:131380] pass2 [obj class] returns MyObject(0x106520120)
2016-03-07 17:19:09.705 testOC[3925:131380] pass3 [obj class] returns MyObject(0x106520120)
2016-03-07 17:19:09.705 testOC[3925:131380] pass4 [obj class] returns MyObject(0x106520120)
2016-03-07 17:19:09.705 testOC[3925:131380] Now recursively call 'object_getClass'
2016-03-07 17:19:09.706 testOC[3925:131380] pass1 object_getClass(obj) returns MyObject(0x106520120)
2016-03-07 17:19:09.706 testOC[3925:131380] pass2 object_getClass(obj) returns MyObject(0x1065200f8)
2016-03-07 17:19:09.706 testOC[3925:131380] pass3 object_getClass(obj) returns NSObject(0x106d7b198)
2016-03-07 17:19:09.712 testOC[3925:131380] pass4 object_getClass(obj) returns NSObject(0x106d7b198)
2016-03-07 17:19:09.712 testOC[3925:131380] Now add KVO to myobj.
2016-03-07 17:19:09.713 testOC[3925:131380] After KVO. [myobj class] returns MyObject(0x106520120)
2016-03-07 17:19:09.713 testOC[3925:131380] After KVO. object_getClass(myobj) returns NSKVONotifying_MyObject(0x7fc66b61f000)
2016-03-07 17:19:09.713 testOC[3925:131380] KVO title changed:Hello, this is new title!
2016-03-07 17:19:09.713 testOC[3925:131380] Now remove KVO of myobj.
2016-03-07 17:19:09.714 testOC[3925:131380] KVO removed. [myobj class] returns MyObject(0x106520120)
2016-03-07 17:19:09.714 testOC[3925:131380] KVO removed. object_getClass(myobj) returns MyObject(0x106520120)
```

##总结：
苹果的`isa-swizzling`（也就是`class-swizzling`）会更改对象的`isa`指针，并且会重写`class`方法以隐藏class已经改变的事实。。。

如果有兴趣，这还一篇文档教你怎么自己实现KVO：
[如何自己动手实现 KVO](http://tech.glowing.com/cn/implement-kvo/)

在搜索的时候，搜到了stackoverflow上类似的一个问题，然而点赞最多的回答并不准确，我在后面附上了准确的回答：
[object_getClass(obj) and [obj class] give different results](http://stackoverflow.com/questions/15906130/object-getclassobj-and-obj-class-give-different-results/35037484#35037484)

##附注释后代码
现在`RACSwizzleClass`的代码就可以读得通了，附上注释如下：
```
// file NSObject+RACSelectorSignal.m
static Class RACSwizzleClass(NSObject *self) {
	Class statedClass = self.class; // 声明的类
	Class baseClass = object_getClass(self); // 实际指向的类
	NSString *className = NSStringFromClass(baseClass);

	if ([className hasSuffix:RACSubclassSuffix]) {
		return baseClass; // 已经swizzle过了
	} else if (statedClass != baseClass) {
    // 正如下面的注释，该对象估计被类似KVO之类的isa-swizzle过了，只能swizzle它的ForwardInvocation方法了

		// If the class is already lying about what it is, it's probably a KVO
		// dynamic subclass or something else that we shouldn't subclass
		// ourselves.
		//
		// Just swizzle -forwardInvocation: in-place. Since the object's class
		// was almost certainly dynamically changed, we shouldn't see another of
		// these classes in the hierarchy.
		//
		// Additionally, swizzle -respondsToSelector: because the default
		// implementation may be ignorant of methods added to this class.
		@synchronized (swizzledClasses()) {
			if (![swizzledClasses() containsObject:className]) {
				RACSwizzleForwardInvocation(baseClass);
				RACSwizzleRespondsToSelector(baseClass);
				[swizzledClasses() addObject:className];
			}
		}

		return baseClass;
	}

  // 没有isa-swizzle过的，直接isa-swizzle
	const char *subclassName = [className stringByAppendingString:RACSubclassSuffix].UTF8String;
	Class subclass = objc_getClass(subclassName);

	if (subclass == nil) {
		subclass = [RACObjCRuntime createClass:subclassName inheritingFromClass:baseClass];
		if (subclass == nil) return nil;

		RACSwizzleForwardInvocation(subclass);
		RACSwizzleRespondsToSelector(subclass);
		objc_registerClassPair(subclass);
	}

	object_setClass(self, subclass);
	return subclass;
}
```

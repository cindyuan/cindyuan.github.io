---
layout: post
title: iOS中runtime 使用小结
subtitle: iOS中runtime 使用小结
date: 2018-03-17
author: xuemin
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
 - iOS
 - Runtime
---

>一直在家带孩子，也没自己弄过博客，面试过两家公司，发现以前的一些知识点都忘了，现在整理下吧。

# Runtime在iOS中的常用方法
- 实现分类（Category）的属性
- 更换代码的实现方法
- 动态增加方法


# 给分类增加属性

大家都知道，分类中是不能声明属性的，即便声明了，也无法调用，因为在分类中，编译器不会自动帮你实现setter和getter方法。

比如我们创建一个`UIImageView`的分类`UIImageView+CateDemo`
```
#import <UIKit/UIKit.h>

@interface UIImageView (CateDemo)

@property (nonatomic, strong) NSString *imgURL;

@end
```

```
#import "UIImageView+CateDemo.h"
#import <objc/runtime.h>

@implementation UIImageView (CateDemo)

- (NSString *)imgURL
{
    return objc_getAssociatedObject(self, @selector(imgURL));
}

- (void)setImgURL:(NSString *)imgURL
{
    objc_setAssociatedObject(self, @selector(imgURL), imgURL, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

@end
```
我们来看看关联属性的这几个方法：
```
OBJC_EXPORT void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0);
    
OBJC_EXPORT id objc_getAssociatedObject(id object, const void *key)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0);
    
OBJC_EXPORT void objc_removeAssociatedObjects(id object)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0);
```
`objc_setAssociatedObject`用于给对象添加关联对象，传nil可以移除相关的关联对象。参数如下：
- object:属性关联
- key:区分属性的唯一标识，因为关联的属性可能不止一个
- value:关联的属性值
- policy:设置关联对象的copy、story、nonatomic等参数，这些常量对应着引用关联值的政策，也就是 Objc 内存管理的引用计数机制。
```
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1,              
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,                                                  
    OBJC_ASSOCIATION_RETAIN = 01401,       
    OBJC_ASSOCIATION_COPY = 01403                                               
};
```

`objc_getAssocicatedObject`用于获取关联对象的值。

`objc_removeAssociatedObject`用于移除该对象的所有关联对象。如果打算只移除一部分则不能使用该方法。

# 更换代码的实现方法
此处需要用到Method Swizzling，其本质的更换了select的IMP

```

@implementation ViewController


+ (void)load
{
    Method classA_method = class_getInstanceMethod([ClassA class], @selector(methodAOfClassAWithArg:));
    Method classB_method = class_getInstanceMethod([ClassB class], @selector(methodAOfClassBWithArg:));

    method_exchangeImplementations(classA_method, classB_method);
}

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _classA = [[ClassA alloc] init];
    _classB = [[ClassB alloc] init];
    [_classA methodAOfClassAWithArg:@"classA 发出的 A方法"];
    [_classB methodAOfClassBWithArg:@"classB 发出的 A方法"];
    
}
```

输出
```
2018-03-17 15:44:38.564 SchedulePrj[4486:634989] methodAOfClassB arg = classA 发出的 A方法
2018-03-17 15:44:38.564 SchedulePrj[4486:634989] methodAOfClassA arg = classB 发出的 A方法
```
首先交换方法写在 +(void)load,在程序的一开始就调用执行，你将不会碰到并发问题。

我们可以发现两个方法的实现过程以及对换。

当然，平时使用我们并不会这么做，当我们要在系统提供的方法上再扩充功能时(不能重写系统方法)，就可以使用Method Swizzling.


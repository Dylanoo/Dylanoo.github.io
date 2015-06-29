---
layout: post
title:  Objective-C中method swizzling技术
date:   2015-06-09 14:17:11
category: "Objective-C"
---



在没有一个类的实现源码的情况下，想改变其中一个方法的实现，除了继承它重写、和借助类别重名方法暴力抢先之外，还有更加灵活的方法吗？在Objective-C编程中，如何实现hook呢？
本文主要介绍针对selector的hook，主角被标题剧透了———— Method Swizzling 。

###Method Swizzling 原理

在Objective-C中调用一个方法，其实是向一个对象发送消息，查找消息的唯一依据是selector的名字。利用Objective-C的动态特性，可以实现在运行时偷换selector对应的方法实现，达到给方法挂钩的目的。
每个类都有一个方法列表，存放着selector的名字和方法实现的映射关系。IMP有点类似函数指针，指向具体的Method实现。

###Method Swizzling 实践
我们来hook一下UINavigationController的pop和push方法
1.首先建立一个UINavigationController的category，并添加对应UINavigationController中的所有pop和push方法，如下：
{% highlight Objective-C %} 
//
//  UINavigationController+Swizzling.h
//  MethodSwizzlling
//
//  Created by Dylan.Lee on 15/6/29.
//  Copyright © 2015年 Dylan.Lee. All rights reserved.
//

#import <UIKit/UIKit.h>

@interface UINavigationController (Swizzling)
/**The method replaced the method of UINavigationViewController
*
- (void)pushViewController:(UIViewController *)viewControllerToShow animated:(BOOL)flag;
*
***/
- (void)cm_pushViewController:(UIViewController *)viewControllerToShow animated:(BOOL)flag;
/**The method replaced the method of UINavigationViewController
*
- (NSArray *)popToRootViewControllerAnimated:(BOOL)flag;
*
***/
- (NSArray *)cm_popToRootViewControllerAnimated:(BOOL)flag;
/**The method replaced the method of UINavigationViewController
*
- (NSArray *)cm_popToViewController:(UIViewController *)viewControllerToShow animated:(BOOL)flag;
*
***/

- (NSArray *)cm_popToViewController:(UIViewController *)viewControllerToShow animated:(BOOL)flag;
/**The method replaced the method of UINavigationViewController
*
- (UIViewController *)cm_popViewControllerAnimated:(BOOL)flag;
*
***/
- (UIViewController *)cm_popViewControllerAnimated:(BOOL)flag;
@end
{% endhighlight %}
需要注意的是，你所写用来hook UINavigationViewController系统方法的方法返回数据类型要跟系统方法保持一致。
在.m文件中实现
{% highlight Objective-C %} 
#import "UINavigationController+Swizzling.h"

@implementation UINavigationController (Swizzling)
- (void)cm_pushViewController:(UIViewController *)viewControllerToShow animated:(BOOL)flag
{
UIViewController *currentVC = nil;
UIViewController *preVC  = nil;
if ([[self.viewControllers lastObject] isKindOfClass:[UIViewController class]])
{
preVC = (UIViewController *)[self.viewControllers lastObject];
}
if ([viewControllerToShow isKindOfClass:[UIViewController class]])
{
currentVC = (UIViewController *)viewControllerToShow;
}
NSLog(@"currentVC.title = %@,preVC.title = %@",currentVC.title,preVC.title);
[self cm_pushViewController:viewControllerToShow animated:flag];
}

- (NSArray *)cm_popToRootViewControllerAnimated:(BOOL)flag
{
//    [self cm_popToRootViewControllerAnimated:flag];
UIViewController *currentVC = nil;
UIViewController *preVC  = nil;
if ([self.viewControllers[0] isKindOfClass:[UIViewController class]])
{
currentVC = (UIViewController *)self.viewControllers[0];
}
if ([[self.viewControllers lastObject] isKindOfClass:[UIViewController class]])
{
preVC = (UIViewController *)[self.viewControllers lastObject];
}
if (currentVC != nil &&preVC != nil)
{
//        currentVC.clikModelDic.previousPage = preVC.clikModelDic.pageName;
}
NSLog(@"currentVC.title = %@,preVC.title = %@",currentVC.title,preVC.title);
return  [self cm_popToRootViewControllerAnimated:flag];


}

- (NSArray *)cm_popToViewController:(UIViewController *)viewControllerToShow animated:(BOOL)flag
{
UIViewController *currentVC = nil;
UIViewController *preVC = nil;
if ([[self.viewControllers lastObject] isKindOfClass:[UIViewController class]])
{
preVC = (UIViewController *)[self.viewControllers lastObject];
}
if ([viewControllerToShow isKindOfClass:[UIViewController class]]&&viewControllerToShow != nil)
{
currentVC = (UIViewController *)viewControllerToShow;
}
NSLog(@"currentVC.title = %@,preVC.title = %@",currentVC.title,preVC.title);
return [self cm_popToViewController:currentVC animated:flag];
}

- (UIViewController *)cm_popViewControllerAnimated:(BOOL)flag
{
UIViewController *currentVC = nil;
UIViewController *preVC = nil;

if ( [self viewControllers].count >=2)
{
NSInteger previousViewControllerIndex = [self viewControllers].count - 2;
if ([[self.viewControllers objectAtIndex:previousViewControllerIndex] isKindOfClass:[UIViewController class]])
{
currentVC = [self.viewControllers objectAtIndex:previousViewControllerIndex];

}
if ([[self.viewControllers lastObject] isKindOfClass:[UIViewController class]])
{
preVC = [self.viewControllers lastObject];
}
NSLog(@"currentVC.title = %@,preVC.title = %@",currentVC.title,preVC.title);

}
return  [self cm_popViewControllerAnimated:flag];

}
@end
{% endhighlight %}
在didFinishLaunchingWithOptions中需要将UINavigationController的方法替换为你所写的方法
{% highlight Objective-C %} 
#import "MethodSwizzingMode.h"
#import <objc/runtime.h>
@implementation MethodSwizzingMode
+(void)setUpMethodSwizzing
{
//UINavigationViewController
Method pushMethod = class_getInstanceMethod([UINavigationController class], @selector(pushViewController:animated:));
Method myPushMethod = class_getInstanceMethod([UINavigationController class], @selector(cm_pushViewController:animated:));
method_exchangeImplementations(pushMethod, myPushMethod);
Method popPreMethod = class_getInstanceMethod([UINavigationController class], @selector(popViewControllerAnimated:));
Method myPopPreMethod = class_getInstanceMethod([UINavigationController class], @selector(cm_popViewControllerAnimated:));
method_exchangeImplementations(popPreMethod, myPopPreMethod);
Method popToRootMethod = class_getInstanceMethod([UINavigationController class], @selector(popToRootViewControllerAnimated:));
Method myPopToRootMethod = class_getInstanceMethod([UINavigationController class], @selector(cm_popToRootViewControllerAnimated:));
method_exchangeImplementations(popToRootMethod, myPopToRootMethod);
Method popMethod = class_getInstanceMethod([UINavigationController class], @selector(popToViewController:animated:));
Method mypopMethod = class_getInstanceMethod([UINavigationController class], @selector(cm_popToViewController:animated:));
method_exchangeImplementations(popMethod, mypopMethod);
}
@end
{% endhighlight %}
这样对于UINavigationController的pop和push方法的hook 就完成了，依照此方法可对UIViewController的present 和dismiss 方法进行hook，这里就不详细介绍了

#[下载demo](https://github.com/Dylan-Lee/MehtodSwizzling/tree/master)
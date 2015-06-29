---
layout: post
title:  非Objc文件(c, c++)引起的NSObjCRuntime错误 
date:   2015-06-29  12:33:11
category: "Objective-C"
---

 今天在objc工程中，导入一些c、c++文件时，，编译引起了NSObjCRuntime错误，仔细检查发现原来是在SK_Prefix.pch中，定义了
{% highlight Objective-C %}
#ifdef __OBJC__
     #import <Foundation/Foundation.h>
     #import <UIKit/UIKit.h>
 #endif
 
     #import "Utils.h"
     #import "Constants.h"
     #import "SKBackgroundNavigation.h"
     #import "BusConfig.h"
{% endhighlight %}

这样导致了Project里的非Objc文件也引入了这些声明，于是出现了上面的错误。修正的办法就是把相关声明都放到__OBJC__里面
{% highlight Objective-C %}
#ifdef __OBJC__ 
 #import <Foundation/Foundation.h> 
 #import <UIKit/UIKit.h> 
 #import "Utils.h"
 #import "Constants.h" 
 #import "SKBackgroundNavigation.h" 
 #import "BusConfig.h" 
#endif
{% endhighlight %}
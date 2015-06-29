---
layout: post
title:  关于Objective-C 中NSTimer的一点深入认识
date:   2015-06-09 14:17:11
category: "Objective-C"
---

到现在做iOS已经差不多两年了，最近才发现自己一直认为用起来十分简单的NSTimer其实并没有那么简单。下面我从几个方面简单介绍下NSTimer的不简单之处：

###系统提供的NSTimer实例化方法:
{% highlight Objective-C %} 


+ (NSTimer *)timerWithTimeInterval:(NSTimeInterval)ti invocation:(NSInvocation *)invocation repeats:(BOOL)yesOrNo;
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti invocation:(NSInvocation *)invocation repeats:(BOOL)yesOrNo;

+ (NSTimer *)timerWithTimeInterval:(NSTimeInterval)ti target:(id)aTarget selector:(SEL)aSelector userInfo:(id)userInfo repeats:(BOOL)yesOrNo;
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti target:(id)aTarget selector:(SEL)aSelector userInfo:(id)userInfo repeats:(BOOL)yesOrNo;

{% endhighlight %}

具体用法这里就不介绍了，需要注意的是用+ (NSTimer *)scheduledTimerWithTimeInterval...初始化的NSTimer会默认以NSDefaultRunLoopMode 为mode 添加到当前的RunLoop ([NSRunLoop currentRunLoop])中，这在Apple的文档中也有注明：

{% highlight Objective-C %} 
+ scheduledTimerWithTimeInterval:invocation:repeats:
Creates and returns a new NSTimer object and schedules it on the current run loop in the default mode.
{% endhighlight %}

除此之外的其他两个方法则需手动将NSTimer对象添加到一个RunLoop中，例如：

{% highlight Objective-C %} 
NSTimer *timer = [NSTimer timerWithTimeInterval:1.0
                                         target:self
                                       selector:@selector(timeHandle:)
                                       userInfo:nil
                                        repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
{% endhighlight %}
否则你会发现你的timer并不能触发目标函数.

### 关于NSTimer释放需要注意的问题:
对于NSTimer的释放，不能像其他对象一样，在-(void)dealloc 中调用[timer release]释放。对于设置了repeats:YES的timer而言，在释放前必须先保证timer对象先停止，并从它所在的runLoop中移除，需要调用apple提供的[timer invalidate] 方法。再在-(void)dealloc中release.而且要保证[timer invalidate] 必须在-(void)dealloc调用之前执行。例如在一个UIViewController中，可在-(void)viewWillDisappear:(BOOL)animated中先停止 timer，再在-(void)dealloc中释放NSTimer的实例。在释放timer之前，必须对timer 进行置空。对于没有没有repeat的timer，如果你的程序在调用- (void)dealloc之前，timer所执行的selector的已经完成，则在- (void)dealloc之前不需调用[timer invalidate]，因为没有repeat的timer 在调用完selector后会自动调用invalidate。
如果你的timer在-(void)dealloc 调用前没有invalidate，你的- (void)dealloc将不会调用，换言之，timer所在类的实例，并没有及时释放。如果你的timer设置了在某个特定场景invalidate，并且你的timer在它所在的类的实例将要销毁时没有invalidate，你会发现，当你- (void)dealloc并没有被调用，而是过了一段时间（也就是timer出发你设置的特定场景invalidate时）- (void)dealloc才会被调用。显然，这并不是我们想要的结果。
### NSTimer 所在runloop的mode 问题
在开发过程中，经常会遇到这样一个需求：把一个UILabel放在一个UIScrollView当中用来显示倒计时，细心的人会发现，很多时候，我在滚动ScrollView的时候，我用来显示倒计时的label的text会停止变化，当我停止scrollview的时候，label会恢复倒计时。What the fuck!!!这是个多么操蛋的bug。
好吧，有问题总要解决的，不能给别人挖坑吧！经过查阅多方资料，终于问题定在timer所在runloop的mode上。
{% highlight Objective-C %} 
//将timer以NSDefaultRunLoopMode添加到currentRunLoop中
//方法一
timer = [NSTimer timerWithTimeInterval:1.0
                                target:self
                              selector:@selector(timeHandle:)
                              userInfo:nil
                               repeats:NO];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
//方法二
timer = [NSTimer scheduledTimerWithTimeInterval:10
                                         target:self
                                       selector:@selector(timeHandle:)
                                       userInfo:nil
                                        repeats:YES];

{% endhighlight %}
上面两种实例话timer的方法，效果是一样的，都是将timer以forMode:NSDefaultRunLoopMode添加到主线程的。当我们用这种发放初始化timer时，你就会发现上面那个操蛋的bug。这是为什么呢？
这是因为timer添加的时候，我们需要指定一个mode，因为同一线程的runloop在运行的时候，任意时刻只能处于一种mode。所以只能当程序处于这种mode的时候，timer才能得到触发事件的机会。对于上面的场景而言，也就是当我们启动timer时，而这时恰好scrollview没有滚动，这是label的显示是正常的。此时主线程的runloopmode也正好是NSDefaultRunLoopMode。当你滚动scrollview时，当前主线程的runloopmode发生变化，变成UITrackingRunLoopMode，别问我怎么知道的，三行代码搞定
{% highlight Objective-C %} 
- (void)timeHandle:(NSTimer *)mtimer
{
  secod -- ;
  NSLog(@"%@",[NSRunLoop currentRunLoop].currentMode);
  self.showLabel.text = [NSString stringWithFormat:@"剩余：%d",secod];
}

-(void)scrollViewDidScroll:(nonnull UIScrollView *)scrollView
{
   NSLog(@"%@",[NSRunLoop currentRunLoop].currentMode);
}
{% endhighlight %}
所以问题就是出在这里了，当我滚动scrollView时，主线程的runloopMode变成了UITrackingRunLoopMode，和timer在主线程中的runloopMode不一致。顺藤摸瓜，那我们改称一样的不就好了，于是就有了下面的代码：
{% highlight Objective-C %} 
//将timer以NSDefaultRunLoopMode添加到currentRunLoop中
//方法一
timer = [NSTimer timerWithTimeInterval:1.0
                                target:self
                                selector:@selector(timeHandle:)
                                userInfo:nil
                                    repeats:NO];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:UITrackingRunLoopMode];
{% endhighlight %}
再滚动ScrollView，label的倒计时正常了～，哈哈哈哈。。。。等等，貌似有什么不对劲，为什么我的label只有滚动的时候才变化?FUCK!!好吧，看看文档The mode set while tracking in controls takes place. You can use this mode to add timers that fire during tracking.意思大概是在拖动loop或其他user interface tracking loops时处于此种模式下，在此模式下会限制输入事件的处理。好吧，这个mode在这个场景是不能用的，经过一番努力（翻阅文档），我找到了这个NSRunLoopCommonModes,文档中的描述是这样的Objects added to a run loop using this value as the mode are monitored by all run loop modes that have been declared as a member of the set of “common" modes。意思大概是这是一个伪模式，其为一组run loop mode的集合，将输入源加入此模式意味着在Common Modes中包含的所有模式下都可以处理。在Cocoa应用程序中，默认情况下Common Modes包含default modes,modal modes,event Tracking modes.可使用CFRunLoopAddCommonMode方法想Common Modes中添加自定义modes。
把mode改为NSRunLoopCommonModes，问题解决～～
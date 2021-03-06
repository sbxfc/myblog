---
layout: post
title: "iOS - Run Loop"
date: 2015-05-08 14:14:25 +0800
comments: true
categories: 
---

#什么是RunLoop

RunLoop是iOS里线程的一部分,任何线程,包括主线程都包含了一个Run Loop对象。RunLoop的作用相当于在线程上维持一个类似while的死循环,使线程在任何时刻都处于待命状态,并且在不执行任务时 RunLoop 会让线程进入睡眠状态(不占用CPU资源)。

主线程上的RunLoop在App运行时由系统自动启动,子线程里的RunLoop需要手动运行。运行时,RunLoop在需要监听Time Source或者Input Source类型的至少一种事件源,如果RunLoop没有绑定事件源,在运行后会立即退出。当事件源没有触发时,RunLoop会让线程进入睡眠状态,当事件源发生时RunLoop会唤醒线程来处理事件。

#运行RunLoop

1）RunLoop 在 Foundation 层和 Core Foundation 层都有对应的接口。

Foundation层:
	
	//运行 RunLoop 在NSDefaultRunLoopMode模式下
	- (void)run;
	//运行 RunLoop 在指定模式下,直到指定的时间结束
	- runMode:beforeDate:
	//运行 RunLoop 在NSDefaultRunLoopMode模式下,直到指定的时间结束
	- runUntilDate:
	- acceptInputForMode:beforeDate:

Core Foundation层:
    
    //运行 CFRunLoopRef
    void CFRunLoopRun();
    //运行 CFRunLoopRef: 参数为运行模式、时间和是否在处理Input Source后退出标志，返回值是exit原因
	SInt32 CFRunLoopRunInMode (mode, seconds, returnAfterSourceHandled);
	//停止运行 CFRunLoopRef
	void CFRunLoopStop( CFRunLoopRef rl );
	//唤醒 CFRunLoopRef
	void CFRunLoopWakeUp ( CFRunLoopRef rl );
  
2）RunLoop的退出方式:

- 以超时配置RunLoop启动,RunLoop会在指定的时间自动退出。

- 显式的停止RunLoop（调用CFRunLoopStop函数）

- <https://developer.apple.com/library/prerelease/mac/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/>  
- <https://developer.apple.com/library/mac/documentation/CoreFoundation/Reference/CFRunLoopRef/>

#运行模式

RunLoop运行时只能以一种固定的模式运行,只会监控这个Mode下添加的Time Source和Input Source。如果这个模式下没有添加事件源,RunLoop会立刻退出。大多数时候，Run Loop都是运行在系统定义的默认模式上。

1）NSDefaultRunLoopMode : 大多数工作默认的运行模式

2) UITrackingRunLoopMode : 用于跟踪触摸事件触发的模式（比如UITableView上下滑动）, 主线程中当触摸事件发生时会设置为这个模式。

3) GSEventReceiveRunLoopMode : 用来接受系统事件,内部的Run Loop模式。

4）NSConnectionReplyMode : 使用这个Mode去监听NSConnection对象的状态，我们很少需要自己使用这个Mode。

5）NSModalPanelRunLoopMode : 使用这个Mode在Model Panel情况下去区分事件(OS X开发中会遇到)。

6) NSRunLoopCommonModes : 是一组常用的模式集合，将一个input source关联到这个模式集合上，等于将input source关联到这个模式集合中的所有模式上。在iOS系统中NSRunLoopCommonMode包含NSDefaultRunLoopMode、NSTaskDeathCheckMode、UITrackingRunLoopMode，我有个timer要关联到这些模式上，一个个注册很麻烦，我可以用

	CFRunLoopAddCommonMode([[NSRunLoop currentRunLoop] getCFRunLoop],(__bridge CFStringRef) NSEventTrackingRunLoopMode)

将NSEventTrackingRunLoopMode或者其他模式添加到这个NSRunLoopCommonModes模式中，然后只需要将Timer关联到NSRunLoopCommonModes，即可以实现Run Loop运行在这个模式集合中任何一个模式时，这个Timer都可以被触发。默认情况下NSRunLoopCommonModes包含了NSDefaultRunLoopMode和UITrackingRunLoopMode。注意：让Run Loop运行在NSRunLoopCommonModes下是没有意义的，因为一个时刻Run Loop只能运行在一个特定模式下，而不可能是个模式集合。


#事件源

归根结底，Run Loop就是个处理事件的Loop，可以添加Timer和其他Input Source等各种事件源，如果事件源没有发生时，Run Loop就可能让线程进入asleep状态，而事件源发生时就会唤醒休眠的(asleep)的子线程来处理事件。Run Loop的事件源事件源分两类：Timer Source和Input Source(包括-performSelector:*API调用簇，Port Input Source、自定义Input Source)。

1）Timer Source类型的事件源,就是创建Timer添加到RunLoop中。在Cocoa里,使用NSTimer创建定时器加入Run Loop,在Core Foundation里使用CFRunLoopTimerRef类型,本质上NSTimer是CFRunLoopTimerRef的简单扩展。需要注意的是,使用scheduledTimerWithTimeInterval 创建的定时器会默认以NSDefaultRunLoopMode模式添加到Run Loop里。而使用timerWithTimeInterval或其他接口创建的Timer需要手动使用 -addTimer:forMode 添加到RunLoop中。如果事件源没有被添加到RunLoop中,则NSTimer不会监听到相应的事件。

	NSTimer* timer = [NSTimer timerWithTimeInterval:1.0f target:self selector:@selector(doFireTimer:) userInfo:nil repeats:YES];
    [runLoop addTimer:timer forMode:NSDefaultRunLoopMode];

2) Input Source中的-performSelector:*API调用簇方法，有以下这些接口：

	performSelectorOnMainThread:withObject:waitUntilDone:  
	performSelectorOnMainThread:withObject:waitUntilDone:modes:
	
	performSelector:onThread:withObject:waitUntilDone:  
	performSelector:onThread:withObject:waitUntilDone:modes:
	
	performSelector:withObject:afterDelay:  
	performSelector:withObject:afterDelay:inModes:
	
	cancelPreviousPerformRequestsWithTarget:  
	cancelPreviousPerformRequestsWithTarget:selector:object:

这些API最后两个是取消当前线程中调用，其他API是在主线程或者当前线程下的Run Loop中执行指定的@selector。

3）Port Input Source

Cocoa和Core Foundation提供了基于端口的对象用于线程或进程间的通信。

如果要建立和<font color="#bd260d">**NSMachPort**</font>对象的本地连接,你需要创建端口对象并加入主线程的Run Loop里。当运行次线程的时候,你传递端口对象到次线程的入口点。次线程通过端口对象将消息传入主线程。

首先在主线程建立端口对象,并在次线程的启动时将端口对象传入:

    //设置主线程port，子线程通过此端口发送消息给主线程
    NSPort *myPort = [NSMachPort port];
    if (myPort) {
        myPort.delegate = self;
        [[NSRunLoop currentRunLoop] addPort:myPort forMode:NSDefaultRunLoopMode];
        
        //启动次线程,并传入端口信息
        [NSThread detachNewThreadSelector:@selector(launchThreadWithPort:) toTarget:[MyWorkerClass class] withObject:myPort];
    }
	


PS:<font color="#bd260d">**NSPortMessage**</font>
是私有 API,官网示例中的线程通实例,客户端实现会有问题。线程间通信完全可以用<font color="#bd260d">**GCD**</font>
或<font color="#bd260d">**performSelector**</font>函数等更简便的方式去实现:
 
<http://stackoverflow.com/questions/12384210/is-nsportmessage-in-the-ios-api>

#观察者

为了方便理解,贴一下代码:

- <https://github.com/sbxfc/RunLoops/>

Run Loop里有一组用于监听事件源的Observers,observers可以通过Core Foundation层的接口定义,并监听指定模式下的事件源触发。

	CFRunLoopAddObserver(cfRunLoop, observer, kCFRunLoopDefaultMode);
	
程序里,大多数情况都是运行在 kCFRunLoopDefaultMode 模式下的,假如我们此时拖动示例中的TableView发现NSTimer事件源不再触发,并且收到RunLoop推出信号。因为此时,RunLoop
	
加入我们监听主线程运行的RunLoop UITrackingRunLoopMode 模式下,而NSTimer默认是运行在 kCFRunLoopDefaultMode 模式下的。有一种解决方式是将Timer添加到forMode:NSRunLoopCommonModes模式下,这也就意味着timer要跟触摸事件同时分享主线程,为了使程序拖动时流畅,在一些很繁重的计算情况下不建议这样去做,因为这样在一定程序上妨碍了流畅度。

        [[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];







    








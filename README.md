#如何解决循环引用的问题 有几种方式
#三种定时器 注意事项
#如何自动管理内存
#NSProxy了解多少

1. 如何解决循环引用的问题？

循环引用通常发生在两个对象相互强引用时，导致内存无法释放。以下是几种常见的解决方案：

方式 1：使用 weak 弱引用

适用场景：delegate、block 等场景。

@property (nonatomic, weak) id<MyDelegate> delegate;
weak 不会增加引用计数，对象释放后自动置为 nil。
方式 2：使用 __weak 打破 Block 中的循环引用

适用场景：Block 中捕获 self 时。
__weak typeof(self) weakSelf = self;
self.block = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    [strongSelf doSomething];
};
__weak 避免 Block 强引用 self，__strong 确保在 Block 执行期间 self 不会被释放。
方式 3：手动断开循环引用

适用场景：对象生命周期明确时。

- (void)dealloc {
    self.otherObject = nil; // 手动断开引用
}
方式 4：使用 NSProxy 或中间对象

适用场景：复杂场景下解耦对象关系。

使用 NSProxy 或中间对象作为代理，避免直接相互引用。
2. 三种定时器及注意事项

定时器 1：NSTimer

特点：
基于 RunLoop，需要添加到 RunLoop 中才能运行。
默认会强引用 target，容易导致循环引用。
注意事项：
使用 weak 打破循环引用：

__weak typeof(self) weakSelf = self;
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
    [weakSelf doSomething];
}];
在 dealloc 中手动释放定时器：

- (void)dealloc {
    [self.timer invalidate];
    self.timer = nil;
}
定时器 2：CADisplayLink

特点：
基于屏幕刷新频率（通常 60Hz），适合 UI 动画。
同样会强引用 target。
注意事项：
使用 weak 打破循环引用：

__weak typeof(self) weakSelf = self;
self.displayLink = [CADisplayLink displayLinkWithTarget:weakSelf selector:@selector(updateUI)];
[self.displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
在 dealloc 中手动释放：

- (void)dealloc {
    [self.displayLink invalidate];
    self.displayLink = nil;
}
定时器 3：GCD 定时器

特点：
基于 dispatch_source，不依赖 RunLoop，精度高。
不会强引用 target，无循环引用问题。

dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_main_queue());
dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC, 0);
dispatch_source_set_event_handler(timer, ^{
    [self doSomething];
});
dispatch_resume(timer);
self.timer = timer;
注意事项：
需要手动管理定时器的生命周期：

if (self.timer) {
    dispatch_source_cancel(self.timer);
    self.timer = nil;
}
3. 如何自动管理内存？

ARC（Automatic Reference Counting）

原理：
编译器在编译时自动插入 retain、release 和 autorelease 调用。
通过引用计数管理对象生命周期。
规则：
strong：强引用，增加引用计数。
weak：弱引用，不增加引用计数，对象释放后自动置为 nil。
copy：复制对象，生成新的不可变对象。
注意事项：
避免循环引用（使用 weak 或 __weak）。
避免在 Block 中捕获 self 导致循环引用。
Autorelease Pool

作用：
延迟释放对象，减少内存峰值。
使用场景：
循环中创建大量临时对象时。

@autoreleasepool {
    for (int i = 0; i < 10000; i++) {
        NSString *tempString = [NSString stringWithFormat:@"%d", i];
        // 使用 tempString
    }
}
4. NSProxy 了解多少？

NSProxy 的作用

定位：
NSProxy 是一个抽象基类，用于实现代理模式或消息转发。
通常用于实现中间对象，解耦对象关系。
特点：
不继承自 NSObject，但实现了 NSObject 协议。
本身不实现任何方法，主要用于消息转发。
使用场景

实现消息转发：
通过 forwardInvocation: 和 methodSignatureForSelector: 实现消息转发。

@interface MyProxy : NSProxy
@property (nonatomic, weak) id target;
@end

@implementation MyProxy
- (void)forwardInvocation:(NSInvocation *)invocation {
    if ([self.target respondsToSelector:invocation.selector]) {
        [invocation invokeWithTarget:self.target];
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    return [self.target methodSignatureForSelector:sel];
}
@end
解决循环引用：
使用 NSProxy 作为中间对象，避免直接相互引用。

@interface MyProxy : NSProxy
@property (nonatomic, weak) id target;
@end

@implementation MyProxy
- (void)forwardInvocation:(NSInvocation *)invocation {
    if ([self.target respondsToSelector:invocation.selector]) {
        [invocation invokeWithTarget:self.target];
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    return [self.target methodSignatureForSelector:sel];
}
@end
实现延迟加载：
使用 NSProxy 延迟加载实际对象，直到第一次调用方法。

@interface LazyProxy : NSProxy
@property (nonatomic, strong) Class targetClass;
@property (nonatomic, strong) id target;
@end

@implementation LazyProxy
- (instancetype)initWithClass:(Class)targetClass {
    self.targetClass = targetClass;
    return self;
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    if (!self.target) {
        self.target = [[self.targetClass alloc] init];
    }
    [invocation invokeWithTarget:self.target];
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    if (!self.target) {
        self.target = [[self.targetClass alloc] init];
    }
    return [self.target methodSignatureForSelector:sel];
}
@end


总结

解决循环引用：
使用 weak、__weak、手动断开引用或 NSProxy。
定时器注意事项：
NSTimer 和 CADisplayLink 需注意循环引用，GCD 定时器更安全。
自动管理内存：
使用 ARC 和 Autorelease Pool 管理内存。
NSProxy：
用于消息转发、解决循环引用和实现延迟加载。

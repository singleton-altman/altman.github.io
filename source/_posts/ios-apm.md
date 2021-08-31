---
title: iOS 应用性能管理SDK建设
tags:
  - APM
  - iOS
  - 性能指标
  - 个人信息安全
date: 2021-08-31 16:14:43
---

**APM**
Application Performance Management，应用性能管理. 随着APP体量越来越大、业务越来越多~~什么功能都要做~~, CPU/内存占用过高、偶尔掉帧、启动缓慢、手机发热...这些问题接踵而来. 随着APP发布安装之后, CPU/内存使用率、帧数是否稳定、启动时间, 都留在了用户的使用过程中. 作为一个合格的开发者, 我们需要把这些数据量化, 展现在自己的面前, 然后分析代码, 解决可能存在的问题, 进而优化体验. 
虽然说市面上已经有相关SDK的服务商, 但是基本上都是黑盒的, 遇到了问题只能提交工单, 然后等待解诀, 但是得到的回复大多都是升级SDK.....所以就自己调研一下吧, 毕竟了解一下也是好的, 是吧?
<!-- more -->
**分析**
近期对市面上的APM相关SDK和服务商做了一些调研, 像`umeng`. SDK主要提供以下几个指标: 
- 崩溃分析
- 卡顿分析
- 启动分析
- 内存分析
- CPU占用率
有了相应的指标, 剩下的就是代码捕获到对应的数据了...再次感谢🙏滴滴团队开源. [doreamentKit](https://github.com/didi/DoraemonKit)
**实施**
- [崩溃分析](#excetion)
    - 监听`signal`, 来自于 `DoraemonCrashSignalExceptionHandler`. 注意⚠️: 除了 `SIGABRT`, 同时还需要监听 `SIGSEGV`、`SIGFPE`、`SIGBUS`、`SIGTRAP`、`SIGILL`、`SIGPIPE`、`SIGSYS`.
```
+ (void)registerHandler {
    // 一定要记录原有的监听, 不然会影响其他相似功能的正常使用, 比如 `umeng`, `bugly`.
    [self backupOriginalHandler];
    // 注册新的监听
    [self signalRegister];
}

+ (void)backupOriginalHandler {
    struct sigaction old_action_abrt;
    sigaction(SIGABRT, NULL, &old_action_abrt);
    if (old_action_abrt.sa_sigaction) {
        previousABRTSignalHandler = old_action_abrt.sa_sigaction;
    }
    ...
}

+ (void)signalRegister {
    DoraemonSignalRegister(SIGABRT);
    ...
}

static void DoraemonSignalRegister(int signal) {
    struct sigaction action;
    action.sa_sigaction = DoraemonSignalHandler;
    action.sa_flags = SA_NODEFER | SA_SIGINFO;
    sigemptyset(&action.sa_mask);
    sigaction(signal, &action, 0);
}

static void DoraemonSignalHandler(int signal, siginfo_t* info, void* context) {
    NSMutableString *mstr = [[NSMutableString alloc] init];
    [mstr appendString:@"Signal Exception:\n"];
    [mstr appendString:[NSString stringWithFormat:@"Signal %@ was raised.\n", signalName(signal)]];
    [mstr appendString:@"Call Stack:\n"];
    // 这里过滤掉第一行日志
    // 因为注册了信号崩溃回调方法，系统会来调用，将记录在调用堆栈上，因此此行日志需要过滤掉
    for (NSUInteger index = 1; index < NSThread.callStackSymbols.count; index++) {
        NSString *str = [NSThread.callStackSymbols objectAtIndex:index];
        [mstr appendString:[str stringByAppendingString:@"\n"]];
    }
    
    [mstr appendString:@"threadInfo:\n"];
    [mstr appendString:[[NSThread currentThread] description]];
    
    // 保存崩溃日志到沙盒cache目录
    ...
    
    DoraemonClearSignalRigister();
    
    // 调用之前崩溃的回调函数
    previousSignalHandler(signal, info, context);
    
    kill(getpid(), SIGKILL);
}

static NSString *signalName(int signal) {
    NSString *signalName;
    switch (signal) {
        case SIGABRT:
            signalName = @"SIGABRT";
            break;
        ...
        default:
            break;
    }
    return signalName;
}

#pragma mark Previous Signal

static void previousSignalHandler(int signal, siginfo_t *info, void *context) {
    SignalHandler previousSignalHandler = NULL;
    switch (signal) {
        case SIGABRT:
            previousSignalHandler = previousABRTSignalHandler;
            break;
        default:
            break;
    }
    
    if (previousSignalHandler) {
        previousSignalHandler(signal, info, context);
    }
}

static void DoraemonClearSignalRigister() {
    signal(SIGABRT,SIG_DFL);
    ...
}
```
    - Exception拦截, 来自`DoraemonUncaughtExceptionHandler`
```
+ (void)registerHandler {
    // 这里也需要提前捕获其他监听, 理由同上
    previousUncaughtExceptionHandler = NSGetUncaughtExceptionHandler();
    
    NSSetUncaughtExceptionHandler(&DoraemonUncaughtExceptionHandler);
}

// 崩溃时的回调函数
static void DoraemonUncaughtExceptionHandler(NSException * exception) {
    // 异常的堆栈信息
    NSArray * stackArray = [exception callStackSymbols];
    // 出现异常的原因
    NSString * reason = [exception reason];
    // 异常名称
    NSString * name = [exception name];
    
    NSString * exceptionInfo = [NSString stringWithFormat:@"========uncaughtException异常错误报告========\nname:%@\nreason:\n%@\ncallStackSymbols:\n%@", name, reason, [stackArray componentsJoinedByString:@"\n"]];
    
    // 保存崩溃日志到沙盒cache目录
    ...
    // 调用之前崩溃的回调函数
    if (previousUncaughtExceptionHandler) {
        previousUncaughtExceptionHandler(exception);
    }
    // 杀掉程序，这样可以防止同时抛出的SIGABRT被SignalException捕获
    kill(getpid(), SIGKILL);
}
```
- 卡顿分析
    - fps, 使用 `CADdisplayLink`
```
- (void)start{
    _link = [CADisplayLink displayLinkWithTarget:self selector:@selector(trigger:)];
    [_link addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
}

- (void)trigger:(CADisplayLink *)link{
    if (_lastTime == 0) {
        _lastTime = link.timestamp;
        return;
    }
    
    _count++;
    NSTimeInterval delta = link.timestamp - _lastTime;
    if (delta < 1) return;
    _lastTime = link.timestamp;
    CGFloat fps = _count / delta;
    _count = 0;
    
    NSInteger intFps = (NSInteger)(fps+0.5);
    self.fps = intFps;
    if (self.block) {
        self.block(self.fps);
    }
}
```
    - 自定义一个ping thread, 定时启动检查是否存在卡顿问题.
```
//  PingThread.m
- (void)main {
    //判断是否需要上报
    __weak typeof(self) weakSelf = self;
    void (^ verifyReport)(void) = ^() {
        __strong typeof(weakSelf) strongSelf = weakSelf;
        // 没有信息不上报, reportInfo 在后面记录
        if (strongSelf.reportInfo.length > 0) {
            if (strongSelf.handler) {
                double responseTimeValue = [[NSDate date] timeIntervalSince1970];
                double duration = (responseTimeValue - strongSelf.startTimeValue)*1000;
                // 上报卡顿间隔
                ...
            }
            strongSelf.reportInfo = @"";
        }
    };
    
    while (!self.cancelled) {
        // 首先把主线程假设为卡顿
        self.mainThreadBlock = YES;
        self.reportInfo = @"";
        self.startTimeValue = [[NSDate date] timeIntervalSince1970];
        // 然后如果这段代码可以执行的话, 那么主线程就不是卡顿状态.
        dispatch_async(dispatch_get_main_queue(), ^{
            self.mainThreadBlock = NO;
            // 检查一下是否需要上报数据
            verifyReport();
            // 传入的信号量dsema的值加1
            dispatch_semaphore_signal(self.semaphore);
        });
        // 当前线程开始休眠
        [NSThread sleepForTimeInterval:self.threshold];
        // 如果上面主线程的代码没有执行, 那么可以得知主线程已经卡顿, 为`reportInfo`赋值.
        if (self.isMainThreadBlock) {
            // 获取主线程的 backtrace
            NSDictionary backtraceOfMainThread = ...;
            self.reportInfo = backtraceOfMainThread;
        }
        // 传入的信号量dsema的值减1，然后等待5s.
        // 如果传入信号量的值等于0，函数将持续等待不返回
        // 如果等待期间没有获取到信号量或者信号量的值一直为0，那么等到timeout时，其所处线程自动执行其后语句。
        dispatch_semaphore_wait(self.semaphore, dispatch_time(DISPATCH_TIME_NOW, 5.0 * NSEC_PER_SEC));
        // 卡顿超时 5.0 * NSEC_PER_SEC 情况;
        { 
            verifyReport();
        }
    }
}
```

- 启动分析
APP启动时间从`main` 函数执行到 `applegate` 中的`- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions`为止. 所以我们选择了一个单例类的`+(void)load`到`- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions`即可. 
```
+ (void)load{
    // hook `- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions`
    ...

    startTime = [[NSDate date] timeIntervalSince1970];
}

- (BOOL)hook_application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions{
    endTime = [[NSDate date] timeIntervalSince1970];
    // (endTime-startTime) 即为启动时间
    return [self hook_application:application didFinishLaunchingWithOptions:launchOptions];
}

```

- 内存分析
```
+ (NSInteger)useMemoryForApp{
    task_vm_info_data_t vmInfo;
    mach_msg_type_number_t count = TASK_VM_INFO_COUNT;
    kern_return_t kernelReturn = task_info(mach_task_self(), TASK_VM_INFO, (task_info_t) &vmInfo, &count);
    if(kernelReturn == KERN_SUCCESS)
    {
        int64_t memoryUsageInByte = (int64_t) vmInfo.phys_footprint;
        return (NSInteger)(memoryUsageInByte/1024/1024);
    }
    else
    {
        return -1;
    }
}
//设备总的内存
+ (NSInteger)totalMemoryForDevice{
    return (NSInteger)([NSProcessInfo processInfo].physicalMemory/1024/1024);
}
```

- CPU占用率
```
+ (CGFloat)cpuUsageForApp {
    kern_return_t kr;
    thread_array_t         thread_list;
    mach_msg_type_number_t thread_count;
    thread_info_data_t     thinfo;
    mach_msg_type_number_t thread_info_count;
    thread_basic_info_t basic_info_th;
    
    // get threads in the task
    //  获取当前进程中 线程列表
    kr = task_threads(mach_task_self(), &thread_list, &thread_count);
    if (kr != KERN_SUCCESS)
        return -1;

    float tot_cpu = 0;
    
    for (int j = 0; j < thread_count; j++) {
        thread_info_count = THREAD_INFO_MAX;
        //获取每一个线程信息
        kr = thread_info(thread_list[j], THREAD_BASIC_INFO,
                         (thread_info_t)thinfo, &thread_info_count);
        if (kr != KERN_SUCCESS)
            return -1;
        
        basic_info_th = (thread_basic_info_t)thinfo;
        if (!(basic_info_th->flags & TH_FLAGS_IDLE)) {
            // cpu_usage : Scaled cpu usage percentage. The scale factor is TH_USAGE_SCALE.
            //宏定义TH_USAGE_SCALE返回CPU处理总频率：
            tot_cpu += basic_info_th->cpu_usage / (float)TH_USAGE_SCALE;
        }
        
    } // for each thread
    
    // 注意方法最后要调用 vm_deallocate，防止出现内存泄漏
    kr = vm_deallocate(mach_task_self(), (vm_offset_t)thread_list, thread_count * sizeof(thread_t));
    assert(kr == KERN_SUCCESS);
    
    if (tot_cpu < 0) {
        tot_cpu = 0.;
    }
    
    return tot_cpu;
}
```

**数据上报机制**
以上可以获取各种指标数据, 下面就要进行上报. 有以下几个节点可以考虑组合使用. 需要注意的是, 上报过程需要线程安全. 不然一边写入, 一边上报会出现各种不可预见的异常或者数据错误.
- 在`applicationWillTerminate`的时候, 不能大量上报, 可能会数据丢失
- 在上文[**崩溃分析**](#excetion)中上报, 同样不建议大量数据同时上传
- 在 `applicationWillEnterForeground` 和 `applicationWillResignActive` 时候上报
- 定时 1 min/2 min或者其他时间节点上报.

**用户个人信息安全**⚠️
最近`《个人信息保护法》`已经颁布, 国内关于个人信息安全相关的关注程度已经不亚于欧盟. 所以作为开发者和供应商一定要切记遵守法规. 因为我们的 APM 相关内容必然涉及到各种用户设备信息, 包括但不仅限于: `udid/openudid`、 `idfa/idfv`、 `wifi`信息、自定义的`设备指纹`等等, 所以**一定**需要在`SDK` **启动/收集 前** 告知用户所有收集的用户/设备相关信息. 如果有可能的话, 一定要让数据`脱敏`, 强烈建议不要上报 `UserID`.

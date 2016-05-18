# iOS崩溃 捕获异常处理

　　网上基本使用的都是同一个版本的异常捕获，我能了解到的关于signal异常捕获的方法也是通过这个版本。我在自己理解的基础上对于这个版本进行了一些修改，也添加了一些注释。下面贴出主要的代码。  

```objective-c
/*!
 *  异常的处理方法
 *
 *  @param install   是否开启捕获异常
 *  @param showAlert 是否在发生异常时弹出alertView
 */
+ (void)installUncaughtExceptionHandler:(BOOL)install showAlert:(BOOL)showAlert {
    
    if (install && showAlert) {
        [[self alloc] alertView:showAlert];
    }
    
    NSSetUncaughtExceptionHandler(install ? HandleException : NULL);
    signal(SIGABRT, install ? SignalHandler : SIG_DFL);
    signal(SIGILL, install ? SignalHandler : SIG_DFL);
    signal(SIGSEGV, install ? SignalHandler : SIG_DFL);
    signal(SIGFPE, install ? SignalHandler : SIG_DFL);
    signal(SIGBUS, install ? SignalHandler : SIG_DFL);
    signal(SIGPIPE, install ? SignalHandler : SIG_DFL);
}
```
　　产生上述的signal的时候就会调用我们定义的`SignalHandler `来处理异常，而`NSSetUncaughtExceptionHandler `就是iOS SDK中提供的一个现成的函数,用来捕获异常的方法，使用方便。但它不能捕获抛出的signal，所以定义了`SignalHandler `方法。  


```objective-c
void HandleException(NSException *exception) {
    
	int32_t exceptionCount = OSAtomicIncrement32(&UncaughtExceptionCount);
    // 如果太多不用处理
	if (exceptionCount > UncaughtExceptionMaximum) {
		return;
	}
	
	//获取调用堆栈
    NSArray *callStack = [exception callStackSymbols];
    NSMutableDictionary *userInfo = [NSMutableDictionary dictionaryWithDictionary:[exception userInfo]];
	[userInfo setObject:callStack forKey:UncaughtExceptionHandlerAddressesKey];
	
    //在主线程中，执行指定的方法, withObject是执行方法传入的参数
	[[[UncaughtExceptionHandler alloc] init]
		performSelectorOnMainThread:@selector(handleException:)
		withObject:
			[NSException exceptionWithName:[exception name]
                         reason:[exception reason]
                         userInfo:userInfo]
        waitUntilDone:YES];
}
```
　　这部分的方法就是对应`NSSetUncaughtExceptionHandler`的处理，只要方法关联到这个函数，那么发生相应错误时会自动调用该函数，调用时会传入`exception`参数。获取异常后会将捕获的异常传入最终调用处理的`handleException`函数，后面会提到。  

```objective-c
//处理signal报错
void SignalHandler(int signal) {
    
	int32_t exceptionCount = OSAtomicIncrement32(&UncaughtExceptionCount);
    // 如果太多不用处理
	if (exceptionCount > UncaughtExceptionMaximum) {
		return;
	}
    
    NSString* description = nil;
    switch (signal) {
        case SIGABRT:
            description = [NSString stringWithFormat:@"Signal SIGABRT was raised!\n"];
            break;
        case SIGILL:
            description = [NSString stringWithFormat:@"Signal SIGILL was raised!\n"];
            break;
        case SIGSEGV:
            description = [NSString stringWithFormat:@"Signal SIGSEGV was raised!\n"];
            break;
        case SIGFPE:
            description = [NSString stringWithFormat:@"Signal SIGFPE was raised!\n"];
            break;
        case SIGBUS:
            description = [NSString stringWithFormat:@"Signal SIGBUS was raised!\n"];
            break;
        case SIGPIPE:
            description = [NSString stringWithFormat:@"Signal SIGPIPE was raised!\n"];
            break;
        default:
            description = [NSString stringWithFormat:@"Signal %d was raised!",signal];
    }
    
	NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
	//获取调用堆栈
	NSArray *callStack = [UncaughtExceptionHandler backtrace];
	[userInfo setObject:callStack forKey:UncaughtExceptionHandlerAddressesKey];
    [userInfo setObject:[NSNumber numberWithInt:signal] forKey:UncaughtExceptionHandlerSignalKey];
    
    //在主线程中，执行指定的方法, withObject是执行方法传入的参数
    [[[UncaughtExceptionHandler alloc] init]
     performSelectorOnMainThread:@selector(handleException:)
     withObject:
     [NSException exceptionWithName:UncaughtExceptionHandlerSignalExceptionName
                  reason: description
                  userInfo: userInfo]
      waitUntilDone:YES];
}
```
　　上面就是用来处理`NSSetUncaughtExceptionHandler`无法捕获的`signal`。由于`signal`传入时都是`int`值，所以我把比较常见的`signal`对应的宏定义在这里描述出来，方便记录和阅读。和上面一样，这里最后也是会将捕获的异常传入`handleException`函数。可以注意到这里获取调用堆栈的方法和上面不同，上面的函数在系统调用时就会传入`exception `，通过`exception `可以很方便的获取到调用堆栈，但是这里不一样，系统调用时传入的仅仅是`signal`值。

```objective-c
//获取调用堆栈
+ (NSArray *)backtrace {
    
    //指针列表
	 void* callstack[128];
    //backtrace用来获取当前线程的调用堆栈，获取的信息存放在这里的callstack中
    //128用来指定当前的buffer中可以保存多少个void*元素
    //返回值是实际获取的指针个数
	 int frames = backtrace(callstack, 128);
    //backtrace_symbols将从backtrace函数获取的信息转化为一个字符串数组
    //返回一个指向字符串数组的指针
    //每个字符串包含了一个相对于callstack中对应元素的可打印信息，包括函数名、偏移地址、实际返回地址
	 char **strs = backtrace_symbols(callstack, frames);
	 
	 int i;
	 NSMutableArray *backtrace = [NSMutableArray arrayWithCapacity:frames];
	 for (i = 0; i < frames; i ++) {
         
	 	[backtrace addObject:[NSString stringWithUTF8String:strs[i]]];
	 }
	 free(strs);
	 
	 return backtrace;
}
```
　　上面就是我们自己获取到调用堆栈的方法。backtrace是Linux下用来追踪函数调用堆栈以及定位段错误的函数。  

```objective-c
- (void)handleException:(NSException *)exception {
    
    [self validateAndSaveCriticalApplicationData:exception];
	
	//不显示alertView就不执行下面的代码
    if (!showAlertView) {
        return;
    }

//alertView 
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wdeprecated-declarations"
	UIAlertView *alert =
		[[UIAlertView alloc]
			initWithTitle:@"出错啦"
			message:[NSString stringWithFormat:@"你可以尝试继续操作，但是应用可能无法正常运行.\n"]
			delegate:self
			cancelButtonTitle:@"退出"
			otherButtonTitles:@"继续", nil];
	[alert show];
#pragma clang diagnostic pop
	
	CFRunLoopRef runLoop = CFRunLoopGetCurrent();
	CFArrayRef allModes = CFRunLoopCopyAllModes(runLoop);
	
	while (!self.dismissed) {
        //点击继续
		for (NSString *mode in (NSArray *)allModes) {
            //快速切换Mode 
			CFRunLoopRunInMode((CFStringRef)mode, 0.001, false);
		}
	}
	
    //点击退出
	CFRelease(allModes);

	NSSetUncaughtExceptionHandler(NULL);
	signal(SIGABRT, SIG_DFL);
	signal(SIGILL, SIG_DFL);
	signal(SIGSEGV, SIG_DFL);
	signal(SIGFPE, SIG_DFL);
	signal(SIGBUS, SIG_DFL);
	signal(SIGPIPE, SIG_DFL);
	
	if ([[exception name] isEqual:UncaughtExceptionHandlerSignalExceptionName]) {
        
		kill(getpid(), [[[exception userInfo] objectForKey:UncaughtExceptionHandlerSignalKey] intValue]);
        
	} else {
        
		[exception raise];
	}
}
```
　　　上面的函数就是最红用来处理异常的函数，在`validateAndSaveCriticalApplicationData`函数里我们可以根据自己的需求进行操作，比如可以把异常信息写入本地在特定的时间发送给指定服务器，或者实时的进行信息的发送等。这里屏蔽了一些警告，因为项目要支持到iOS7。

## 使用
```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {    
    // Override point for customization after app launch    
	
    [UncaughtExceptionHandler installUncaughtExceptionHandler:YES showAlert:YES];

	return YES;
}
```

####写在最后
　　其实这上面的代码我也不是完全弄懂了每一句话，比如`OSAtomicIncrement32(&UncaughtExceptionCount)`，比如alertView点击了退出按钮后执行的一系列代码。如果有知道的留言告诉我一下吧~~小女子不胜感激！
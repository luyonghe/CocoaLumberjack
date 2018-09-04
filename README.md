# CocoaLumberjack

CocoaLumberjack 是一个适用于 Mac 和 iOS 的快速、简单，而且功能强大且灵活的日志框架

## 一、功能介绍

1. 支持日志打印到 Xcode 控制台，打印到 mac 的 console 程序、打印到文件
2. 日志功能支持关闭和打开，支持 Error、Warning、Info、Debug、Verbose、All 日志等级
3. 支持基于 DDLogFormatter 协议自定义日志格式和继承 DDAbstractLogger 和协议 DDLogger 自定义 logger
4. 可将日志打印上传到服务器
5. 基于多线程、GCD、运行时态等

## 二、架构

由四个部分组成：DDLog, DDLogger, DDLogMessage, DDLogFormatter。

### 2.1 DDLog与DDLogger

![](/15335271789746.png)

如上图所示，DDLog 是整个库的入口，我们平时用的 DDLogLevel 等宏就是直接调用 DDLog 的接口。一个 DDLog 包含着一个或多个 DDLogger，比如常用的DDTTYLogger，DDASLLogger 等等。当我们调用 DDLogInfo 的时候，DDLog 会把已有的 DDLogger 全部遍历一遍，对于每个 DDLogger 都会调用其 logMessage 接口。当然细心的读者会意识到，这些操作不可能同步(SYNC)进行。而且如果要保证日志的相对顺序，必然会给 DDLogger 分配一个专属的线性队列(SERIAL_QUEUE)。

### 2.2 DDLogger，DDLogFormatter 与 DDLogMessage

这三者之间的关系，用图像解释反而更加容易迷惑，还是用文字吧。

先阐述一下整个流程。DDLog 调用 DDLogger 的时候，传入的是一个 DDLogMessage 实例。这个实例除了日志内容本身之外，还包含了关于这个日志的辅助信息，主要包括日志等级(level)，环境（context），队列（queueLabel），标签（tag），时间戳（timestamp）等等，他们的用法下面会有详细介绍。接着，DDLogger 里面如果有 DDLogFormatter 的话，会先调用 DDLogFormatter 处理日志格式，对日志进行过滤等等操作，然后把日志打到 Logger 对应的位置。

这里要注意，每个 DDLogger 最多只能有一个 DDLogFormatter 实例，可以通过 setLogFormatter：设置。但是，如果有需要同时应用多个 DDLogFormatter 的话，可以使用框架内置的一个 DDMultiFormatter 把需要用的整合到一起。这样就使得用户自定义 formatter 非常的方便。这里要注意两点：
（1）DDMultiFormatter 应用 formatter 时，是按照其添加的顺序逐个依次线性地进行的，所以往 DDMultiFormatter 上添加 formatter 时一定要注意添加的顺序；

（2）虽然 DDLogFormatter 名字上叫做 Formatter，它的功能并不局限于改变日志的格式。你还可以用它来过滤日志等等。

### 2.3 DDLogMessage 相关属性
DDLogMessage 属性的作用是给 DDLogFormatter 足够的信息去修改日志格式以及进行过滤筛选等操作。这里贴一下 DDLogMessage 的属性列表：
```objectivec
@interface DDLogMessage : NSObject <NSCopying>
{
    // Direct accessors to be used only for performance
    @public
    NSString *_message;
    DDLogLevel _level;
    DDLogFlag _flag;
    NSInteger _context;
    NSString *_file;
    NSString *_fileName;
    NSString *_function;
    NSUInteger _line;
    id _tag;
    DDLogMessageOptions _options;
    NSDate *_timestamp;
    NSString *_threadID;
    NSString *_threadName;
    NSString *_queueLabel;
}
```
大部分属性都相对比较容易理解，这里想着重梳理一下 context 和 level。截止至写作日期，Github 上的文档还没有更新，所以如果你去读文档的话，会感到非常混乱。CocoaLumberjack 自从 v2.x.x 以来，社区大幅度地改变了对这两个属性的定义。

按照[文档](https://github.com/CocoaLumberjack/CocoaLumberjack/blob/master/Documentation/CustomLogLevels.md)所说，用户可以通过在自己的头文件里#define自定义日志等级。但是如果你去试试就会发现根本行不通。原因是自从2.x版本以后，日志等级的定义方式由原来的宏改成了枚举(NS_ENUM)。这几乎杜绝了用户自定义日志等级。


## 三、安装

第一种方法：使用 `CocoaPods`，`Podfile` 看起来是这样的：

```objectivec
platform:ios, '7.0'
target 'CocoaLumberjackDemo' do
pod 'CocoaLumberjack'
end
```
第二种方法：使用 `[Carthage](https://github.com/Carthage/Carthage) `， `Cartfile` ：

`github "CocoaLumberjack/CocoaLumberjack"`

第三种方法：手工导入，具体可以看[他的文档](https://github.com/CocoaLumberjack/CocoaLumberjack/blob/master/Documentation/GettingStarted.md#manual-installation)

## 四、使用

**CocoaLumberjack 自带了几种 Log 方式：**

> 1.DDLog（整个框架的基础）
>
> 2.DDASLLogger（发送日志语句到苹果的日志系统，以便它们显示在 Console.app上）
>
> 3.DDTTYLoyger（发送日志语句到Xcode控制台）
>
> 4.DDFIleLoger（把日志写入本地文件

你可以同时记录文件和控制台，还可以创建自己的 logger，将日志语句发送到网络或者数据库中。

使用的时候需要引入头文件：#import <CocoaLumberjack/CocoaLumberjack.h>，你还需要全局设置下 log 级别：static const DDLogLevel ddLogLevel = DDLogLevelDebug;，关于 Log 级别，下面会细讲。

所以你的.pch 里面可能有段这样的代码：

![](/15335275099106.png)

然后加入代码：

```objectivec
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
    // 添加DDASLLogger，你的日志语句将被发送到Xcode控制台
    [DDLog addLogger:[DDTTYLogger sharedInstance]];
    
    // 添加DDTTYLogger，你的日志语句将被发送到Console.app
    [DDLog addLogger:[DDASLLogger sharedInstance]];
    
    // 添加DDFileLogger，你的日志语句将写入到一个文件中，默认路径在沙盒的Library/Caches/Logs/目录下，文件名为bundleid+空格+日期.log。
    DDFileLogger *fileLogger = [[DDFileLogger alloc] init];
    fileLogger.rollingFrequency = 60 * 60 * 24;
    fileLogger.logFileManager.maximumNumberOfLogFiles = 7;
    [DDLog addLogger:fileLogger];
    
    //产生Log
    DDLogVerbose(@"Verbose");
    DDLogDebug(@"Debug");
    DDLogInfo(@"Info");
    DDLogWarn(@"Warn");
    DDLogError(@"Error");
    
    return YES;
}
```

DDLog 和 NSLog 的语法是一样的。

运行程序，可以在 Xocde 控制台看到：

![Xcode 日志](/15335275615889.png)

产生的 Log 文件打开是这样的：

![Log 文件](/15335275741484.png)

**Log 级别**

接下来，你就要考虑用哪种级别了，CocoaLumberjack 有5种：
```objectivec
typedef NS_OPTIONS(NSUInteger, DDLogFlag){
    DDLogFlagError      = (1 << 0),
    DDLogFlagWarning    = (1 << 1),
    DDLogFlagInfo       = (1 << 2),
    DDLogFlagDebug      = (1 << 3),
    DDLogFlagVerbose    = (1 << 4)
};
```
例如，如果您将日志级别设置为 LOG_LEVEL_INFO，那么你会看到error、Warn和Info语句。

你也可以[自定义Log级别及每个级别的名字](https://github.com/CocoaLumberjack/CocoaLumberjack/blob/master/Documentation/CustomLogLevels.md)或者[在单纯的级别上增加一些高级用法](https://github.com/CocoaLumberjack/CocoaLumberjack/blob/master/Documentation/FineGrainedLogging.md)

我们也可以为 Debug 和 Release 模式设置不同的 Log 级别：
```objectivec
#ifdef DEBUG 
static const DDLogLevel ddLogLevel = DDLogLevelVerbose;
#else 
static const DDLogLevel ddLogLevel = DDLogLevelWarning;
#endif
```
我们还可以为每种 loger 设置不同的级别：

```objectivec
[DDLog addLogger:[DDASLLogger sharedInstance] withLevel:DDLogLevelInfo];
[DDLog addLogger:[DDTTYLogger sharedInstance] withLevel:DDLogLevelDebug];
```

我们还可以[自定义日志的 formatter 格式](https://github.com/CocoaLumberjack/CocoaLumberjack/blob/master/Documentation/CustomFormatters.md)：
首先自定义一个 LogFormatter, 遵从 DDLogFormatter 协议，我们需要重写 - (NSString *)formatLogMessage:(DDLogMessage *)logMessage 这个方法，这个方法的输入参数是由 logger 发的一个 DDLogMessage 对象，包含了一些必要的信息，返回值就是最终 log 的消息体字符串。



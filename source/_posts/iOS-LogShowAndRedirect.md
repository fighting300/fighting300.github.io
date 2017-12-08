---
title: iOS中的日志同步获取-NSLog重定向及其他
tags: iOS
date: 2017-12-08 21:13:48
---


我们在真机测试时经常会发现一个难题是无法查看真机的`NSLog`类型的实时日志，这时候需要RD复现问题来定位当时的日志，以方便查找问题。这个问题在测试中是非常常见的，也是功能测试会花费比较长时间的一个原因。以下我们讨论下能即时查看日志的几种方案。

##### NSLog输出到哪里？
在iOS开发中，我们经常会用到NSLog调试，但是我们却不太了解它。在NSLog本质是一个C函数，它的函数声明如下：
`FOUNDATION_EXPORT void NSLog(NSString *format, ...)`
系统对它的说明是：`Logs an error message to the Apple System Log facility.`。他是用来输出信息到标准Error控制台上去的，其内部其实是使用`Apple System Log`的API。在调试阶段，日志会输出到到Xcode中，而在iOS真机上，它会输出到系统的`/var/log/syslog`这个文件中。

在iOS中，把日志输出到文件中的句柄在`unistd.h`文件中有定义：    
```
#define	 STDIN_FILENO	0	/* standard input file descriptor */
#define	STDOUT_FILENO	1	/* standard output file descriptor */
#define	STDERR_FILENO	2	/* standard error file descriptor */
```
NSLog输出的是到`STDERR_FILENO`上，我们可以在iOS中使用c语言输出到的文件的`fprintf`来验证：  
```
NSLog(@"iOS NSLog");
fprintf (stderr, "%s\n", "fprintf log");
```
由于fprintf并不会像NSLog那样，在内部调用ASL接口，所以只是单纯的输出信息，并没有添加日期、进程名、进程id等，也不会自动换行。

<!--more-->
##### ASL读取日志  
首先我们可以想到的是既然日志写入系统的syslog中，那我们可以直接读取这些日志。从ASL读取日志的核心代码如下：  

```
#import <asl.h>
// 从日志的对象aslmsg中获取我们需要的数据
+(instancetype)logMessageFromASLMessage:(aslmsg)aslMessage{
    SystemLogMessage *logMessage = [[SystemLogMessage alloc] init];
    const char *timestamp = asl_get(aslMessage, ASL_KEY_TIME);
    if (timestamp) {
        NSTimeInterval timeInterval = [@(timestamp) integerValue];
        const char *nanoseconds = asl_get(aslMessage, ASL_KEY_TIME_NSEC);
        if (nanoseconds) {
            timeInterval += [@(nanoseconds) doubleValue] / NSEC_PER_SEC;
        }
        logMessage.timeInterval = timeInterval;
        logMessage.date = [NSDate dateWithTimeIntervalSince1970:timeInterval];
    }
    const char *sender = asl_get(aslMessage, ASL_KEY_SENDER);
    if (sender) {
        logMessage.sender = @(sender);
    }
    const char *messageText = asl_get(aslMessage, ASL_KEY_MSG);
    if (messageText) {
        logMessage.messageText = @(messageText);//NSLog写入的文本内容
    }
    const char *messageID = asl_get(aslMessage, ASL_KEY_MSG_ID);
    if (messageID) {
        logMessage.messageID = [@(messageID) longLongValue];
    }
    return logMessage;
}
+ (NSMutableArray<SystemLogMessage *> *)allLogMessagesForCurrentProcess{
    asl_object_t query = asl_new(ASL_TYPE_QUERY);
    // Filter for messages from the current process. Note that this appears to happen by default on device, but is required in the simulator.
    NSString *pidString = [NSString stringWithFormat:@"%d", [[NSProcessInfo processInfo] processIdentifier]];
    asl_set_query(query, ASL_KEY_PID, [pidString UTF8String], ASL_QUERY_OP_EQUAL);

    aslresponse response = asl_search(NULL, query);
    aslmsg aslMessage = NULL;

    NSMutableArray *logMessages = [NSMutableArray array];
    while ((aslMessage = asl_next(response))) {
        [logMessages addObject:[SystemLogMessage logMessageFromASLMessage:aslMessage]];
    }
    asl_release(response);

    return logMessages;
}
```
使用以上方法的好处是不会影响Xcode控制台的输出，可以用非侵入性的方式来读取日志。


##### NSLog重定向  
另一种方式就是重定向NSLog，这样NSLog就不会写到系统的syslog中了。

###### dup2重定向  
通过重定向，可以直接截取`stdout,stderr`等标准输出的信息，然后保存在想要存储的位置，上传到服务器或者显示到View上。  
要做到重定向，需要通过`NSPipe`创建一个管道，pipe有读端和写端，然后通过`dup2`将标准输入重定向到pipe的写端。再通过`NSFileHandle`监听pipe的读端，最后再处理读出的信息。
之后通过printf或者NSLog写数据，都会写到pipe的写端，同时pipe会将这些数据直接传送到读端，最后通过NSFileHandle的监控函数取出这些数据。  
核心代码如下：  
```
- (void)redirectStandardOutput{
    //记录标准输出及错误流原始文件描述符
    self.outFd = dup(STDOUT_FILENO);
    self.errFd = dup(STDERR_FILENO);
#if BETA_BUILD
    stdout->_flags = 10;
    NSPipe *outPipe = [NSPipe pipe];
    NSFileHandle *pipeOutHandle = [outPipe fileHandleForReading];
    dup2([[outPipe fileHandleForWriting] fileDescriptor], STDOUT_FILENO);
    [pipeOutHandle readInBackgroundAndNotify];

    stderr->_flags = 10;
    NSPipe *errPipe = [NSPipe pipe];
    NSFileHandle *pipeErrHandle = [errPipe fileHandleForReading];
    dup2([[errPipe fileHandleForWriting] fileDescriptor], STDERR_FILENO);
    [pipeErrHandle readInBackgroundAndNotify];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(redirectOutNotificationHandle:) name:NSFileHandleReadCompletionNotification object:pipeOutHandle];

    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(redirectErrNotificationHandle:) name:NSFileHandleReadCompletionNotification object:pipeErrHandle];
#endif
}

-(void)recoverStandardOutput{
#if BETA_BUILD
    dup2(self.outFd, STDOUT_FILENO);
    dup2(self.errFd, STDERR_FILENO);
    [[NSNotificationCenter defaultCenter] removeObserver:self];
#endif
}

// 重定向之后的NSLog输出
- (void)redirectOutNotificationHandle:(NSNotification *)nf{
#if BETA_BUILD
    NSData *data = [[nf userInfo] objectForKey:NSFileHandleNotificationDataItem];
    NSString *str = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    // YOUR CODE HERE...  保存日志并上传或展示
#endif
    [[nf object] readInBackgroundAndNotify];
}

// 重定向之后的错误输出
- (void)redirectErrNotificationHandle:(NSNotification *)nf{
#if BETA_BUILD
    NSData *data = [[nf userInfo] objectForKey:NSFileHandleNotificationDataItem];
    NSString *str = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    // YOUR CODE HERE...  保存日志并上传或展示
#endif
    [[nf object] readInBackgroundAndNotify];
}
```

###### 文件重定向
另一种重定向的方式是利用c语言的`freopen`函数进行重定向，将写往stderr的内容重定向到我们制定的文件中去，一旦执行了上述代码那么在这个之后的NSLog将不会在控制台显示了，会直接输出在指定的文件中。
在模拟器中，我们可以使用终端的tail命令(tail -f xxx.log)对这个文件进行实时查看，就如同我们在Xcode的输出窗口中看到的那样，你还可以结合grep命令进行实时过滤查看，非常方便在大量的日志信息中迅速定位到我们要的日志信息。   
`FILE * freopen ( const char * filename, const char * mode, FILE * stream );`

具体代码如下：  
```
   NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
   NSString *documentsPath = [paths objectAtIndex:0];
   NSString *loggingPath = [documentsPath stringByAppendingPathComponent:@"/xxx.log"];
   //redirect NSLog
   freopen([loggingPath cStringUsingEncoding:NSASCIIStringEncoding], "a+", stderr);
```
这样我们就可以把可获取的日志文件发送给服务端或者通过itunes共享出来。但是由于iOS严格的沙盒机制，我们无法知道stderr原来的文件路径，也无法直接使用沙盒外的文件，所以freopen无法重定向回去，只能使用第1点所述的dup和dup2来实现。
```
// 重定向
int origin1 = dup(STDERR_FILENO);
FILE * myFile = freopen([loggingPath cStringUsingEncoding:NSASCIIStringEncoding], "a+", stderr);
// 恢复重定向
dup2(origin1, STDERR_FILENO);
```

###### 使用GCD的dispatch Source重定向方式

具体代码如下：  
```
- (dispatch_source_t)_startCapturingWritingToFD:(int)fd  {
    int fildes[2];
    pipe(fildes);  // [0] is read end of pipe while [1] is write end
    dup2(fildes[1], fd);  // Duplicate write end of pipe "onto" fd (this closes fd)
    close(fildes[1]);  // Close original write end of pipe
    fd = fildes[0];  // We can now monitor the read end of the pipe
    char* buffer = malloc(1024);
    NSMutableData* data = [[NSMutableData alloc] init];
    fcntl(fd, F_SETFL, O_NONBLOCK);
    dispatch_source_t source = dispatch_source_create(DISPATCH_SOURCE_TYPE_READ, fd, 0, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0));
    dispatch_source_set_cancel_handler(source, ^{
        free(buffer);
    });
    dispatch_source_set_event_handler(source, ^{
        @autoreleasepool {

            while (1) {
                ssize_t size = read(fd, buffer, 1024);
                if (size <= 0) {
                    break;
                }
                [data appendBytes:buffer length:size];
                if (size < 1024) {
                    break;
                }
            }
            NSString *aString = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
            //printf("aString = %s",[aString UTF8String]);
            //NSLog(@"aString = %@",aString);
            // Do something
        }
    });
    dispatch_resume(source);
    return source;
}
```

##### 日志同步/上传      
重定向或者存储的数据可以传到服务端或者通过server同步到网页上，就可以更方便的看到这些数据了。
如果想再网页端实时查看日志，可以在App内置一个小型http web服务器。GitHub上开源的项目有[GCDWebServer](https://github.com/swisspol/GCDWebServer)，可以使用该工具，在APP开启webserver服务，并在同一局域网下，使用`http://localhost:8080`来请求最新日志了。
上传服务端的部分很简单，实现简单的网络请求就可以，这儿不做叙述。
另外在实际项目中，可以设置一个开关来开启或关闭这个重定向，在调试测试的过程中可以打开开关来查看程序当前的日志。

通过以上处理，真机测试中，日志就可以很方便的获取和查看了，这样能节省不少人力和时间成本。

###### 参考文档
1. [iOS IO重定向](http://lizaochengwen.iteye.com/blog/1476080)  
2. [iOS 日志](http://www.bijishequ.com/detail/355322?p=)  

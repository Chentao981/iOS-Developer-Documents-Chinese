#解析DNS主机名</br>
[原文地址](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/NetworkingTopics/Articles/ResolvingDNSHostnames.html#//apple_ref/doc/uid/TP40012543-SW1)
翻译人:一帆  日期:2015.9.10 审核人:ibcker 审核日期:2015.10.25

这部分将探讨如何以最灵活地方式解析DNS主机名。

>重要：在OS X和iOS中大多数高层协议的API允许你通过DNS名字来建立连接，同时也建议你这样做。一个主机名可以同时映射到多个IP地址。如果你要自己解析主机名，你需要选择一个IP地址来作为链接远程主机用。作为对照，如果你通过名字来链接，你需要允许操作系统来选择最好的链接方式。尤其对于Bonjour服务更是这样，因为IP地址随时都在变化。
>
>然而，通过主机名来链接并不是总是可以。如果你使用的API必须用IP地址的话，OS X和iOS提供几种方法来获取一个或者多个IP地址根据给定的DNS主机名。这一节会描述这些技术。
>
>在你读下面内容之前，你可以读一下 Avoid Resolving DNS Names Before Connecting to a Host。

在OS X和iOS中有三种API来处理主机名的解析：NSHost(只用于OS X),CFHost和POSIX标准的API。

* NSHost—即使通过NSHost是一个常用方法来解析主机名，但是现在并不推荐用它因为他是一个同步API。也就是说，用它会导致系统系能下降。因为它不推荐使用，这里不会对他讲解。
* CFHost—CFHost API允许你进行异步操作，如果你必须自己解决这些问题，那他是一个解析主机名比较理想的方案。
* POSIX—这个标准提供几个函数来解析主机名。这些方法只有当你需要用于处理跨平台的代码时才用，比如你需要集成你的代码到已有的POSIX标准的网络模块上。

###通过CFHost解析主机名

通过CFHost解析主机名：

1. 通过调用 CFHostCreateWithName 创建一个 CFHostRef 对象。
2. 调用 CFHostSetClient 并且提供一个上下文对象和回调函数，这个回调函数在解析结束的时候会被调用。
3. 调用 CFHostScheduleWithRunLoop 用于执行具体的解析操作，在你的 run loop 中。
4. 调用 CFHostStartInfoResolution 来告诉解析器开始解析，把它的第二个参数设置为 kCFHostAddresses 表明你想要返回一个IP地址。
5. 等待解析器调用你的回调函数，通过你的回调函数，调用 CFHostGetAddressing 函数来获取解析结果。这个函数返回 CFDataRef 对象的一个数组，其中的每一个都包含一个POSIX的`sockaddr`结构体。

反向域名解析的过程(通过IP地址得到主机名)与上面相似，你可以调用 CFHostCreateWithAddress 来创建一个对象，把  kCFHostNames 传入 CFHostStartInfoResolution ，然后调用 CFHostGetNames 来获取结果。

###通过POSIX解析主机名
如果打算通过POSIX调用来解析主机名，你需要知道这些调用是同步的并且不能用在图形应用的主线程上。所以，你需要创建一个单独的POSIX线程或者一个GCD任务并且在其上下文中执行它。

>重要：利用POSIX在iOS应用的主线程中执行主机名解析，如果超时，会导致你的应用被系统终止。

POSIX在`<netdb.h>`文件里面定义了三个方法来解析主机名：

`getaddrinfo`

   * 返回所有已经解析好的地址信息，这个方法是一个非常好的在POSIX标准上获取地址信息的函数，你可以在他的手册上找到一些示例代码.
   
	>重要：一些老的NDS服务器没有遵循IPv6协议会返回一个错误。POISX的`getaddrinfo`函数会尝试隐藏这个错误，通过在成功接收到一个IPv4请求以后取消IPv6请求。当然，如果你的应用需要获取IPv6地址，即使能链接成功(而不是获取一个IPv4地址以后就停止查询)，你应该使用一个异步的API比如CFHost。
	
`gethostbyname`

   * 根据传入的主机名返回IPv4地址。这个方法是不推荐的因为它只能查询IPv4的地址。

`gethostbyname2`

  * 根据传入的主机名，返回在一些指定的协议(AF_INET, AF_INET6等等)下的地址信息。即使这个方法允许你得到IPv4的地址信息就像`gethostbyname`提供的那样，但是它无法一次获取多个地址和选择最快的。所以，这个方法可以考虑为在已经存在的代码的`gethostbyname`方法的一个替代。在新代码中最好使用`getaddrinfo`函数。
  
对于反向域名解析(通过IP地址得到主机名)，POSIX提供 getnameinfo 和 gethostbyaddr 方法。getnameinfo方法是推荐选项因为它更灵活。

要了解更多信息，读取上面的链接中这个模块的主页。

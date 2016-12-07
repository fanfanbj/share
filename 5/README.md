#Gopher

##两个月的Gopher,一点点小兴奋

公司的技术栈是Golang。作为Java工程师，我在用Spring Boot给某客户开发一个proxy Service服务，早已对Golang垂涎三尺，so利用业余时间开始看《Go语言圣经》，因为没有practice，看完后，基本“阅后即焚”。

看到Linus Torvalds对Golang的评价，是否会跟我一样，小兴奋一下呢？Linus能这样说，已经肯定了Golang，只是需要时间来证明。

![image](https://github.com/fanfanbj/share/blob/master/5/image/linus.png)

今年十一国庆假期前，我结束了北京银行的实施工作，在十一期间，阅读了《Go语言圣经》，《Go语言程序设计》，《VIM实用技巧》等书。我选择vim作为我的开发环境，it looks very cool. 十一后，我加入了公司Borgsphere项目组开发，正式开放工作，开发任务是负责应用管理功能增加权限认证。之后，开始了自研项目，发布工具Baker的开发工作。经历了两个月的Golang开发，完全被Golang吸引，it is a amazing。在此，分享几点心得。

###Intercepter模式
深受Java OOP的多年教育，实现应用管理功能增加权限认证，一定会想到Intercepter模式。定义应用读权限认证拦截器和应用写权限认证拦截器。
![image](https://github.com/fanfanbj/share/blob/master/5/image/AppWriterAuthInterceptor.png)
AuthError是自定义的结构体，包括HTTP ResponseCode，内部TransactionCode：TCode，及ErrorMessage。
![image](https://github.com/fanfanbj/share/blob/master/5/image/AuthError.png)

在AppWriterAuthInterceptor中，可以进行更好的函数封装，在后来发布工具baker中，我做了一些enhance的改进。定义了MustAdmin()和MustUser()，路漫漫其修远矣。
![image](https://github.com/fanfanbj/share/blob/master/5/image/user.png)

###TDD的尝试
定义var AppWriterAuthInterceptor = func(acc *auth.Account, application *marathon.Application) *AuthError 最初没有将函数定义为变量。但做单元测试的时候，mock对象遇到一些问题。最后想到简洁的办法是将函数定义为变量。下面是Unit-Test mock对象及一个UT的例子：

![image](https://github.com/fanfanbj/share/blob/master/5/image/ut-1.png)

![image](https://github.com/fanfanbj/share/blob/master/5/image/ut-2.png)

曾看了一篇Pivotal工程师写得关于TDD的文章，之前我们对Unit-Test给予很高的职责：

![image](https://github.com/fanfanbj/share/blob/master/5/image/ut-3.png)

过重的Unit-Test有时会导致投入多出2-3倍的精力在写Unit-Test。有时会冒火，UT降低了敏捷的速度。我们开始争论80％以上覆盖率的UT，如何写得更有效？
我们将Unit-test的职责变成：

![image](https://github.com/fanfanbj/share/blob/master/5/image/ut-4.png)
我们可以遵循下面的原则完成Unit－test，同时可以保持Agile敏捷：


“Every class should be paired with well-designed unit test”

~~“Every class should be paired with well-designed unit test”~~

“Every behavior should be paired with well-designed unit test”

###Golang Web编程
使用Golang ginrus lib写一个Web服务器变成了一件容易的事情。首先定义router，router会定义middleware中间层逻辑，再启动web服务。代码如下：

![image](https://github.com/fanfanbj/share/blob/master/5/image/web-1.png)

在baker项目中，middleware加载UserDB到缓存，配置文件，构建工作池，并加载到http request的context上。下面的代码是初始化构建工作池bakeworkpool，并加载到context上。

![image](https://github.com/fanfanbj/share/blob/master/5/image/web-2.png)

在baker项目中对session的处理，是将登录认证用户的相关信息，如：Token，通过设置Set-cookie头将session的信息（token）传送到客户端,而客户端此后的每一次请求，都会带上这个标识符。并将用户相关信息缓存到缓存区。

	httputil.SetCookie(c.Writer, c.Request, "user_sess", tokenstr)
	
用户请求会带上Session的信息（token）,服务器Session模块会验证token，并将用户信息加载到context上。主要代码如下：

![image](https://github.com/fanfanbj/share/blob/master/5/image/web-3.png)

麻雀虽小，五脏俱全的web服务器开始工作啦。
	
###试试WorkPool
考虑到发布工具并发的考虑，发布工具baker的参考架构图如下：

![image](https://github.com/fanfanbj/share/blob/master/5/image/baker.jpg)

在参考架构图中，baker通过Job Executor处理请求的构建／发布Job, Executor将从context获得缓存的WorkPool，并用Sync.WaitGroup处理Work，一个Work由多个Task组成，当前task是顺序执行。并启动新goroutin collector收集work执行的状态信息。Executor的代码如下：

![image](https://github.com/fanfanbj/share/blob/master/5/image/workpool-2.png)


WorkPool使用了Cloud Foundry的WorkPool,WorkPool结构体中定义带缓冲的channel，存放func(),以此建立一种工作池的机制。WorkPool结构体如下：

![image](https://github.com/fanfanbj/share/blob/master/5/image/workpool-1.png)

代码参见baker项目：https://github.com/Dataman-Cloud/baker




#Gopher

##两个月的Gopher,一点点小兴奋

公司的技术栈是Golang。作为Java工程师，我在用Spring Boot给某客户开发一个proxy Service服务，早已对Golang垂涎三尺，so利用业余时间开始看《Go语言圣经》，因为没有practice，看完后，基本“阅后即焚”。

看到Linus Torvalds对Golang的评价，是否会跟我一样，小兴奋一下呢？Linus能这样说，已经肯定了Golang，只是需要时间来证明。

![image](https://github.com/fanfanbj/share/blob/master/5/linus.png)

今年十一国庆假期前，我结束了北京银行的实施工作，在十一期间，阅读了《Go语言圣经》，《Go语言程序设计》，《VIM实用技巧》等书。我选择vim作为我的开发环境，it looks very cool. 十一后，我加入了公司Borgsphere项目组开发，正式开放工作，开发任务是负责应用管理功能增加权限认证。之后，开始了自研项目，发布工具Baker的开发工作。经历了两个月的Golang开发，完全被Golang吸引，it is a amazing。在此，分享几点心得。

###Intercepter模式
深受Java OOP的多年教育，实现应用管理功能增加权限认证，一定会想到Intercepter模式。定义应用读权限认证拦截器和应用写权限认证拦截器。
![image](https://github.com/fanfanbj/share/blob/master/5/AppWriterAuthInterceptor.png)
AuthError是自定义的结构体，包括HTTP ResponseCode，内部TransactionCode：TCode，及ErrorMessage。
![image](https://github.com/fanfanbj/share/blob/master/5/AuthError.png)

在AppWriterAuthInterceptor中，可以进行更好的函数封装，在后来发布工具baker中，我做了一些enhance的改进。定义了MustAdmin()和MustUser()，路漫漫其修远矣。
![image](https://github.com/fanfanbj/share/blob/master/5/user.png)

###TDD的尝试
定义var AppWriterAuthInterceptor = func(acc *auth.Account, application *marathon.Application) *AuthError 最初没有将函数定义为变量。但做单元测试的时候，mock对象遇到一些问题。最后想到简洁的办法是将函数定义为变量。下面是Unit-Test mock对象及一个UT的例子：

![image](https://github.com/fanfanbj/share/blob/master/5/ut-1.png)

![image](https://github.com/fanfanbj/share/blob/master/5/ut-2.png)

曾看了一篇Pivotal工程师写得关于TDD的文章，之前我们对Unit-Test给予很高的职责：

![image](https://github.com/fanfanbj/share/blob/master/5/ut-3.png)

过重的Unit-Test有时会导致投入多出2-3倍的精力在写Unit-Test。有时会冒火，UT降低了敏捷的速度。我们开始争论80％以上覆盖率的UT，如何写得更有效？
我们将Unit-test的职责变成：

![image](https://github.com/fanfanbj/share/blob/master/5/ut-4.png)

“Every class should be paired with well-designed unit test”

~~“Every class should be paired with well-designed unit test”~~

“Every behavior should be paired with well-designed unit test”




###Golang Web编程
###WorkPool/Goroutin的尝试
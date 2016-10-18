#Too young too simple
##Spring Boot和Docker

初想SpringBoot是做Web应用开发的，内嵌Servlet容器，是一种Web容器。Docker是以应用为单元进行封装隔离。他们的关系就好比Openstack的虚拟网络neutron和基于BGP协议的Calico虚拟网络一样，是不同层次的解决方案。（neutron是Iaas层虚拟网络的解决方案，Calico提供了容器层虚拟网络的解决方案。）Spring Boot，从字面上理解，Boot是引导的意思，他帮助Java开发者快速启动一个Web容器，是Java应用容器化的解决方案。Docker是基于LXC的应用容器引擎的应用容器化技术，标准的Docker容器包含软件组件及其依赖 。类似Calico虚拟网络解决方案可以对接到Openstack虚拟网络上，我们可以在Docker容器中部署运行Spring Boot应用。

这么看来Spring Boot和Docker目标都是应用容器化，Spring Boot是Java开发者的福音，简化了Java开发部署配置及应用监控等工作。Docker做为应用容器引擎，可以独立于硬件、语言、框架，轻视实现持续集成与部署，快速迭代。从跨平台方面，Java是跨平台语言，Docker也发布了Windows Server 2016的windows容器，所以，这个角度看，Spirng Boot和Docker应用容器化，都可以实现应用只需一次构建即可多个平台运行。他们都是颠覆者，都有点Too young, too simple的感觉。:)

下面让我们深入看看Spring Boot和Docker：

##Spring Boot，Web容器
“Java要死了吗”这个问题，随着Spring Boot的出现又一次pending。Spring Boot可以说是近5年来Spring乃至整个Java社区最有影响力的项目之一。Spring cloud更是借着Spring Boot和Cloud Native的东风让整个Java生态系统又一次不甘寂寞，保持着活跃的用户群。
回顾历史，Java做为服务端开发语言一路走来，从最初的Servlet／Jsp，到分布式架构EJB的出现，再到轻量级框架Spring。许多公司开始采用Java做为主流的语言进行企业应用开发。但Java EE使得Spring逐渐变得笨重，大量的XML配置文件存在，繁琐的配置，复杂的远程Debug，及难办的性能监控等问题。
Spring Boot解决了如下问题：
	1. Spring Boot使编码变简单。
	2. Spring Boot使配置变简单。
	3. Spring Boot使部署变简单。
	4. Spring Boot使监控变简单。
Spring Boot内嵌Servlet容器，我们可以使用java -jar proxyservice-0.0.1.jar 运行项目，或者执行项目的主程序main函数，达到快速运行。

Spring Boot支持如下的Servlet容器：

![image](https://github.com/fanfanbj/share/blob/master/2/springboot1.png)

SpringBoot是伴随着Spring4.0诞生，他继承了Spring框架的基因，帮助开发者快速搭建一个Web容器，做为成熟的语言或框架，他真正做到让开发者只关注在业务逻辑实现,框架的事情都由Spring和SpringBoot来完成。

![image](https://github.com/fanfanbj/share/blob/master/2/springboot2.png)

同时，Spring Boot提供了一系列的starters来简化我们的Maven依赖，能够非常方便的进行包管理, 很大程度上减少了jar hell或者dependency hell。

![image](https://github.com/fanfanbj/share/blob/master/2/springboot3.png)

Spring Boot只需要很少的配置，大部分的时候我们直接使用默认的配置即可。Spring Boot会根据我们项目中类路径的jar包/类，为jar包的类进行自动配置Bean，这样一来就大大的简化了我们的配置。当然，这只是Spring考虑到的大多数的使用场景，在一些特殊情况，我们还需要自定义自动配置。

同时，Spring Boot提供了基于http、ssh、telnet对运行时的项目进行监控；这个听起来是不是很炫酷！

示例：以SSH登录为例

1、首先，添加starter pom依赖

	<dependency>
   	 <groupId>org.springframework.boot</groupId>
   	 <artifactId>spring-boot-starter-remote-shell</artifactId>
	</dependency>


2、运行项目,此时在控制台中会出现SSH访问的密码：

![image](https://github.com/fanfanbj/share/blob/master/2/springboot4.png)

3、SSH登录到应用程序，端口为2000，用户为user：

![image](https://github.com/fanfanbj/share/blob/master/2/springboot5.png)


##Docker，应用容器引擎

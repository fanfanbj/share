#SpringBoot和Docker的比较
##Too young,too simple
初想SpringBoot是做Web应用开发的，内嵌Servlet容器，是一种Web容器。Docker是以应用为单元进行封装隔离。他们的关系就好比Openstack的neutron和基于BGP协议的Calico虚拟网络一样，是不同层次的解决方案。（neutron是Iaas层虚拟网络的解决方案，Calico提供了容器层虚拟网络的解决方案。）SpringBoot，从字面上理解，Boot是引导的意思，他帮助Java开发者快速启动一个Web容器，是Java应用容器化的解决方案。Docker是基于LXC的应用容器引擎的应用容器化技术，标准的Docker容器包含软件组件及其依赖 。同时，类似Calico虚拟网络解决方案可以对接Openstack虚拟网络上，我们可以在Docker容器中部署运行SpringBoot应用。

这么看来SpringBoot和Docker目标都是应用容器化，SpringBoot是Java开发者的福音，简化了Java开发部署配置及应用监控等工作。Docker做为应用容器引擎，更具有通用性，可以独立于硬件、语言、框架，实现持续集成与部署，快速迭代。从跨平台方面，Java是跨平台语言，Docker也发布了Windows Server 2016的windows容器，所以，这个角度看，Spirng Boot和Docker应用容器化，都可以实现应用只需一次构建即可多个平台运行。他们都是颠覆者，都有点Too young, too simple的感觉。:)

下面让我们深入看看SpringBoot和Docker：

##SpringBoot，Web容器
“Java要死了吗”这个问题，随着SpringBoot的出现又一次pending。SpringBoot可以说是近5年来Spring乃至整个Java社区最有影响力的项目之一。SpringCloud更是借着SpringBoot和Cloud Native的东风让整个Java生态系统又一次不甘寂寞，保持着活跃的用户群。

回顾历史，Java做为服务端开发语言一路走来，从最初的Servlet／Jsp，到分布式架构EJB的出现，再到轻量级框架Spring。许多公司开始采用Java做为主流的语言进行企业应用开发。但Java EE使得Spring逐渐变得笨重，大量的XML配置文件存在，繁琐的配置，复杂的远程Debug，及难办的性能监控等问题。这些却只是问题的一小部分。

“远古时代”时，运行Java程序，我们需要先运行应用服务器如：Tomcat，Weblogic，之后部署应用，应用服务器帮管理应用的生命周期。随着应用高可用及弹性伸缩需求的增加及应用趋于微服务化，大大增加了运维的复杂度，应用服务器这种“远古”模式的Web容器，已经无法满足需求。SpringBoot的出现，用应用容器取代了传统的Tomcat，Weblogic这样的Web容器，同时，SpringBoot解决了如下问题：

* SpringBoot使编码变简单。
* SpringBoot使配置变简单。
* SpringBoot使部署变简单。
* SpringBoot使监控变简单。

SpringBoot内嵌Servlet容器，我们可以使用java -jar proxyservice-0.0.1.jar 运行项目，或者执行项目的主程序main函数，达到快速运行。

SpringBoot支持如下的Servlet容器：

![image](https://github.com/fanfanbj/share/blob/master/2/springboot1.png)

SpringBoot是伴随着Spring4.0诞生，他继承了Spring框架的基因，帮助开发者快速搭建一个Web容器，做为成熟的语言或框架，他真正做到让开发者只关注在业务逻辑实现,框架的事情都由Spring和SpringBoot来完成。

![image](https://github.com/fanfanbj/share/blob/master/2/springboot2.png)

同时，SpringBoot提供了一系列的starters来简化我们的Maven依赖，能够非常方便的进行包管理, 很大程度上减少了jar hell或者dependency hell。

![image](https://github.com/fanfanbj/share/blob/master/2/springboot3.png)

SpringBoot只需要很少的配置，大部分的时候我们直接使用默认的配置即可。SpringBoot会根据我们项目中类路径的jar包/类，为jar包的类进行自动配置Bean，这样一来就大大的简化了我们的配置。当然，这只是Spring考虑到的大多数的使用场景，在一些特殊情况，我们还需要自定义自动配置。

同时，SpringBoot提供了基于http、ssh、telnet对运行时的项目进行监控；这个听起来是不是很炫酷！

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
下面让我们看看2016年超火的Docker，Docker技术使用Linux的namespace和cgroups对应用进行封装隔离，可以说是一种应用级的虚拟化技术。这些大特征暂且放在一边，个人感觉Docker的Dockerfile和编排功能却是具有革命性，将Docker推到了一个巅峰，CI/CD或是DevOps都有他们的身影。

个人认为Dockerfile和编排功能的革命性在于他们设计的通用性特质。这点通过对比Cloud Foundry的BuildPacks就能看出来。BuildPacks提供了框架和应用运行时的支持，可以根据用户设置来决定应用的依赖和配置。Buildpacks针对不同语言提供不同的系统BuildPacks。而Dockerfile和编排是独立于语言，框架，环境的安装部署。通过生成客户化镜像，定义服务之间依赖的编排文件及通过ENV或ENVfile定义应用服务配置信息。用此方式可以完成在不同环境安装部署。

以SpringCloud为例，看看应用Docker容器化及通过Docker-compose发布整个项目。如下是一个SpringCloud的demo应用：
![image](https://github.com/fanfanbj/share/blob/master/2/docker1.png)
我们可以为每个SpringCloud的服务定义安装部署的Dockerfile文件,并用Docker build生成服务镜像：

![image](https://github.com/fanfanbj/share/blob/master/2/docker2.png)

同时编写编排文件，使用Docker-compose为SpringCloud项目编排文件，是再合适不过的事情了。除此之外，数人云的安装部署过程也使用了Dockerfile和编排，并配合Ansible完成了专业的DevOps安装包。同时，值得一提，SpringCloud中的一些服务健康检查，服务监控，服务调度等功能，又很好的丰富了Docker应用容器的解决方案。

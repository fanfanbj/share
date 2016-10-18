#Too yong too simple
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
Spring Boot内嵌Servlet容器，我们可以使用java -jar proxyservice-0.0.1.jar 运行项目，或者执行项目的主程序main函数，快速运行。Spring Boot支持










##Docker，应用容器引擎
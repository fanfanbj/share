#Docker存储方式实践分享
# 第一部分 问题诊断事情从一次实施项目说起，我们需要帮助客户将他们的应用容器化并在数人云平台上发布应用。客户应用是传统WAS应用。应用是通过WAS console界面进行手工部署，暂时无法通过Dockerfile进行自动化应用部署，最后的镜像是通过Docker commit完成。镜像启动执行命令是startwas.sh,并通过tail应用日志将输出到标准输出。启动容器，WAS Server启动失败，错误日志如下：
![image](http://)

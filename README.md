# LinuxCluster
搭建的Linux高可用集群主要实现以下几个功能：

1、本地yum源，只有一台主机连接外网，所以其它主机全都通过它来安装软件；同时也因此，会用nfs来共享文件；   
2、lvs+keepalived实现web服务的高可用和负载均衡,静态数据由nginx处理，动态数据由apache处理；                      
3、lvs+keepalived+mha+mycat实现mysql数据库的高可用负载均衡和读写分离；                  
4、ELK对集群中的主机进行日志收集和分析；                                                                                                       
5、通过zabbix监控集群中的所有主机；                                                                                                           
6、利用github+jenkins实现自动部署；  


所有用到的软件我都已经放到百度网盘
链接：https://pan.baidu.com/s/1pYMaNUUemhol4lgUBK4l-w 
提取码：l7jr 

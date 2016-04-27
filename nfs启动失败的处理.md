  在centos7里，nfs服务依赖于rpcbind服务（有些linux版本是依赖于portmap服务），故应在rpcbind服务启动成功后再启动nfs服务。貌似在linux的service文件增加依赖关系只能控制两个服务启动的先后次序，不能实现*前一服务启动成功后再启动后一服务*。故应手动启动。
```
systemctl restart rpcbind.service 
systemctl restart nfs
```
Links
-----
 - [My blog](./)



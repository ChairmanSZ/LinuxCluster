一、前期准备：

1、做好ssh免密码登录：

```shell
[lvs01]ssh-keygen -t dsa

ssh-copy-id root@192.168.1.11 ~12 21 22 31 32 41 42 51 52 53 61 
```

2、修改/etc/yum.conf

```shell
keepcache=1   #yum install 的时候保留下载文件，因为其它机子要用

cachedir=/usr/local/repos     #下载文件的存放目录

```

3、修改/etc/hosts文件：

```shell
192.168.1.11    lvs01
192.168.1.12    lvs02
192.168.1.21    nginx01
192.168.1.22    nginx02
192.168.1.31    apache01
192.168.1.32    apache02
192.168.1.41    mycat01
192.168.1.42    mycat02
192.168.1.51    mysql01
192.168.1.52    mysql02
192.168.1.53    mysql03
192.168.1.61    redis01
```

4、创建存放目录

```shell
mkdir /usr/local/repos   #/usr/local/repos作为本地yum仓库
mkdir /usr/local/tools    #创建nfs共享的目录
```

5、安装一些常用工具

```shell
yum install -y net-tools tree lrzsz ansible vim createrepo
```

二、开始安装配置ansible、nginx和nfs

1、安装ansible

```shell
yum install ansible -y

vim /etc/ansible/hosts

#设置好各个主机组
[bros]
lvs02
nginx01
nginx02
apache01
apache02
mycat01
mycat02
mysql01
mysql02
mysql03
redis01

[mycat]
mycat01
mycat02

[mysql]
mysql01
mysql02
mysql03


[lvs]
lvs01
lvs02

[nginx]
nginx01
nginx02

[apache]
apache01
apache02

[redis]
redis01


#ansible 主机名 -m ping   每个都ping一次，确认状态是能ping通

```



2、安装nginx

```shell
yum install -y pcre* openssl* gcc gcc+    
groupadd nginx
useradd -g nginx -s /sbin/nologin -M
cd /usr/local/src
wget http://nginx.org/download/nginx-1.16.1.tar.gz
tar -zxf nginx-1.16.1.tar.gz
cd nginx-1.16.1.tar.gz
/configure --prefix=/usr/local/nginx --user=www --group=www --with-http_stub_status_module --with-http_ssl_module
make
make install 
echo "/usr/local/nginx/sbin/nginx" >> /etc/rc.local
```

3、配置nginx.conf

```shell
vim /usr/local/nginx/conf/nginx.conf
http标签内：
   autoindex  on;
   autoindex_exact_size  on;
   autoindex_localtime  on;

server标签内：
  listen  8080；
  location / {
       root    /usr/local/repos;
       index index.html index.htm;
 }

/usr/local/nginx/sbin/nginx   #启动nginx
```

4、更新yum仓库

```shell
createrepo  /usr/local/repos
```

5、写好repo文件

```shell
vim cluster.repo
[cluster]
name=cluster
enabled=1
baseurl=http://192.168.1.11:8080
gpgcheck=0
```

6、编写分发repo文件的脚本

```shell
vim repo.sh     

#!/bin/bash
for n in 12 21 22 31 32 41 42 51 52 53 61 
do
  scp cluster.repo 192.168.1.$n:/etc/yum.repos.d/
  ssh 192.168.1.$n "cd /etc/yum.repos.d && mv CentOS-Base.repo CentOS-Base.repo.bak && yum clean all && yum makecache"
done

chmod a+x repo.sh

./repo.sh
```

7、安装nfs和rpcbind

```shell
yum install -y nfs-utils rpcbind
vim  /etc/exports
/usr/local/tools 192.168.1.0/24(ro,sync,all_squash)
systemctl start rpcbind
systemctl start nfs
```

8、更新本地yum仓库，让其他机子能安装nfs

```shell
createrepo /usr/local/repos
ansible bros -m shell -a "yum clean all"
ansible bros -m shell -a "yum makecache"
ansible bros -m shell -a "yum install -y nfs-utils"
ansible bros -m shell -a "mkdir /usr/local/tools"
ansible bros -m shell -a "mount -t nfs 192.168.1.11:/usr/local/tools /usr/local/tools"
```



本地yum源和nfs共享存储搭建完成！！！
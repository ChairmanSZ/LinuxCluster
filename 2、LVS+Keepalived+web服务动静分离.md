一、安装apache和php 

#因为只有lvs01有外网ip，所以要通过lvs01来安装   apche服务器:192.168.1.31~32

```shell
[lvs01]   yum install -y httpd --downloadonly
rpm -ivh http://mirrors.ustc.edu.cn/epel/epel-release-latest-7.noarch.rpm  
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
createrepo /usr/local/repos
ansible apache -m shell -a "yum clean all"
ansible apache -m shell -a "yum yum makecache"
ansible apache -m yum -a "name=httpd state=present"
ansible apache -m yum -a "name=php70w state=present"
ansible apache -m shell -a "systemctl start httpd"
ansible apache -m shell -a "systemctl enable httpd"
ansible apache -m shell -a "php -v "   #查看是否安装成功
```

　　

二、安装nginx并配置好动静分离[192.168.1.21和192.168.1.22两部机子都要操作 ]

1、安装nginnx

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

2、配置nginx.conf实现静态数据由nginx处理，动态数据如php让apache处理

```shell
vim /usr/local/nginx/conf/nginx.conf

#http标签内增加upstream，名字叫cluster：
http {
    upstream cluster {
        server  192.168.1.31:80 weight=1;
        server  192.168.1.32:80 weight=1;
    }
｝

#server标签内相关内容修改如下：
server {
    listen       80;
    server_name  localhost;
    location / {
        root   /var/www/html;   #因为apache的网站根目录是/var/www/html,所以这里跟apache保持一致
        index  index.html index.htm;
    }
    location ~ \.php$ {
        proxy_pass  http://cluster;      #如果访问以.php结尾的文件，就交给apache处理
    }
}


```

3、测试动静分离是否成功：

```shell
[nginx服务器]mkdir /var/www/html

cd /var/www/html

rz 1.png   #上传nginx的Logo图片


[apache服务器]cd /var/www/html

vim index.php

<?php

  echo "这是apache服务器";

  echo "<img src='1.png'>";

?>

rz 1.png  #上传apache的Logo图片

curl nginx01/index.php    #如果动静分离成功，能成功看到apache的index.php的内容，但显示的图片是nginx的logo图片，而不是apache的logo图片
```

三、动静分离成功接下来就做负载均衡和高可用 [192.168.1.11和192.168.1.12 两部机子都要操作]    

1、安装ipvsadm

```shell
yum -y install ipvsadm
```

2、安装keepalived

```shell
yum install -y openssl-devel
cd /usr/local/src
wget http://www.keepalived.org/software/keepalived-2.0.16.tar.gz
tar zxf keepalived-2.0.16.tar.gz
cd keepalived-2.0.16
./configrue --prefix=/usr/local/keepalived
make
make install clean
cp /usr/local/src/keepalived-2.0.16/keepalived/etc/init.d/keepalived /etc/init.d/keepalived
chmod +x /etc/init.d/keepalived
echo "/etc/init.d/keepalived start " >> /etc/rc.local
mkdir /etc/keepalived
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
cp /usr/local/keepalived/etc/sbin/keepalived /usr/sbin
```

3、配置keepalived

```shell
vim /etc/keepalived/keepalived.conf

! Configuration File for keepalived

global_defs {
   router_id LVS_01    #lvs02设置为LVS_02
}

vrrp_instance VI_1 {
    state MASTER     #lvs02设置为BACKUP
    interface eth0
    virtual_router_id 51
    priority 100     #lvs02设置为90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.101/24  brd 192.168.1.255 dev eth0 label eth0:1   #虚拟ip设置为192.168.1.101
    }
}

virtual_server 192.168.1.101 80 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 2
    protocol TCP

    real_server 192.168.124.21 80 {
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
    }

    real_server 192.168.124.22 80 {
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
        }
     }
}


systemctl start keepalived    #启动keepalived
```

5、在nginx服务器的lo网卡绑定VIP地址并修改内核参数抑制ARP响应 [192.168.1.21和192.168.1.22两部机子都要操作 ]

```shell
ip addr add 192.168.1.101/32 dev lo

cat >>/etc/sysctl.conf<<EOF
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
EOF
sysctl -p
```

6、测试

华为云的外网ip绑定在虚拟ip192.168.1.101上，然后浏览器访问外网ip，成功访问到动静分离的web服务器！

7、给keepalived高可用服务添加邮件告警功能

转自：https://blog.51cto.com/11886307/2406629 ，个人亲测有效，感谢博主


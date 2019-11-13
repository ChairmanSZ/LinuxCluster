前期规划：

```shell
mysql01     192.168.1.51    （主库   MHA node）

mysql02     192.168.1.52    （从库   MHA node）

mysql03     192.168.1.53   （从库    MHA manager）

mycat01     192.168.1.61

mycat02     192.168.1.62
```

一、安装mysql并做好主从复制：

```shell
#mysql01-03安装过程一致
#卸载系统自带的mariadb
ansible mysql -m shell -a "/bin/rpm -e $(/bin/rpm -qa | grep mariadb|xargs) --nodeps"

[lvs01]yum -y install make gcc-c++ cmake bison-devel ncurses-devel --downloadonly
createrepo /usr/local/repos
ansible mysql -m shell -a "yum clean all"
ansible mysql -m shell -a "yum makecache"
ansible mysql -m shell -a "yum -y install make gcc-c++ cmake bison-devel ncurses-devel "
cd /usr/local/tools
rz boost_1_59_0.tar.gz  mysql-5.7.26.tar.gz
ansible mysql -m copy -a "src=/usr/local/tools/boost_1_59_0.tar.gz dest=/usr/local/src/"
ansible mysql -m copy -a "src=/usr/local/tools/mysql-5.7.26.tar.gz dest=/usr/local/src/"

[mysql01-03]cd /usr/local/src
tar -zxf boost_1_59_0.tar.gz
mv boost_1_59_0 /usr/local/boost

groupadd mysql
useradd -g mysql mysql -M -s /sbin/nologin
cd /usr/local/src
tar -zxvf mysql-5.7.26.tar.gz
cd mysql-5.7.26/

cmake -DCMAKE_INSTALL_PREFIX=/data/mysql -DMYSQL_DATADIR=/data/mysql/data -DSYSCONFDIR=/etc -DWITH_MYISAM_STORAGE_ENGINE=1 -DWITH_INNOBASE_STORAGE_ENGINE=1 -DWITH_MEMORY_STORAGE_ENGINE=1 -DWITH_READLINE=1 -DMYSQL_UNIX_ADDR=/var/lib/mysql/mysql.sock -DMYSQL_TCP_PORT=3306 -DENABLED_LOCAL_INFILE=1 -DWITH_PARTITION_STORAGE_ENGINE=1 -DEXTRA_CHARSETS=all -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci -DWITH_BOOST=/usr/local/boost

make && make install

mkdir -p /data/mysql/data
mkdir /data/mysql/var
chown -R mysql:mysql /data/mysql

#初始化mysql
/data/mysql/bin/mysqld --basedir=/data/mysql --datadir=/data/mysql/data --user=mysql --initialize

#编辑/etc/my.cnf
[client]
port = 3306
socket = /usr/local/mysql/var/mysql.sock
   
[mysqld]
port = 3306
socket = /usr/local/mysql/var/mysql.sock

skip-grant-tables  #设置免密码登录，修改完密码后删掉这一行

basedir = /usr/local/mysql/
datadir = /usr/local/mysql/data
pid-file = /usr/local/mysql/data/mysql.pid
user = mysql
bind-address = 0.0.0.0
server-id = 1     #mysql01-03的sever-id分别为1 2 3
sync_binlog=1
log_bin = mysql-bin
   
skip-name-resolve
#skip-networking
back_log = 600
   
max_connections = 3000
max_connect_errors = 3000
##open_files_limit = 65535
table_open_cache = 512
max_allowed_packet = 16M
binlog_cache_size = 16M
max_heap_table_size = 16M
tmp_table_size = 256M
   
read_buffer_size = 1024M
read_rnd_buffer_size = 1024M
sort_buffer_size = 1024M
join_buffer_size = 1024M
key_buffer_size = 8192M
   
thread_cache_size = 8
   
query_cache_size = 512M
query_cache_limit = 1024M
   
ft_min_word_len = 4
   
binlog_format = mixed
expire_logs_days = 30
   
log_error = /usr/local/mysql/data/mysql-error.log
slow_query_log = 1
long_query_time = 1
slow_query_log_file = /usr/local/mysql/data/mysql-slow.log
   
performance_schema = 0
explicit_defaults_for_timestamp
   
##lower_case_table_names = 1
   
skip-external-locking
   
default_storage_engine = InnoDB
##default-storage-engine = MyISAM
innodb_file_per_table = 1
innodb_open_files = 500
innodb_buffer_pool_size = 4096M
innodb_write_io_threads = 1000
innodb_read_io_threads = 1000
innodb_thread_concurrency = 8
innodb_purge_threads = 1
innodb_flush_log_at_trx_commit = 2
innodb_log_buffer_size = 4M
innodb_log_file_size = 32M
innodb_log_files_in_group = 3
innodb_max_dirty_pages_pct = 90
innodb_lock_wait_timeout = 120
   
bulk_insert_buffer_size = 8M
#myisam_sort_buffer_size = 8M
#myisam_max_sort_file_size = 1G
#myisam_repair_threads = 1
   
interactive_timeout = 28800
wait_timeout = 28800
   
[mysqldump]
quick
max_allowed_packet = 16M
   
[myisamchk]
key_buffer_size = 8M
sort_buffer_size = 8M
read_buffer = 4M
write_buffer = 4M
   
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
port = 3306

#设置mysql开机自启动
cp /data/mysql/support-files/mysql.server /etc/init.d/mysql
chkconfig mysql on
service mysql start

echo "export PATH=$PATH:/data/mysql/bin" >> /etc/profile
source /etc/profile
   
mkdir -p /var/lib/mysql
ln -s /data/mysql/var/mysql.sock /var/lib/mysql/mysql.sock

#登录mysql并修改密码
mysql -uroot -p
flush privileges;
set password for root@'localhost' identified by '123456';
flush privileges;


#开始搭建主从复制
[mysql01]mysql -uroot -p
grant replication slave on *.* to 'root'@'192.168.1.%' identified by '123456';
flush privileges;
show master status;

[mysql02-03]mysql -uroot -p
change master to master_host='192.168.1.51',master_user='root',master_password='123456',master_port=3306,master_log_file='mysql-bin.000001',master_pos=154;
start slave;
show slave status\G
 #出现 Slave_IO_Running: Yes 和 Slave_SQL_Running: Yes  说明主从复制成功

```

 

二、搭建mha高可用集群

1、建立软链接：

```shell
ln -s /data/mysql/bin/mysqlbinlog    /usr/bin/mysqlbinlog
ln -s /data/mysql/bin/mysql          /usr/bin/mysql
```

2、配置mysql各个节点的ssh互信:

```shell
ssh-keygen
ssh-copy-id   192.168.1.51
scp /root/.ssh root@192.168.124.52~53:/root/.ssh
```

3、mysql所有节点安装MHA Node软件

```shell
[lvs01]yum install perl-DBD-MySQL -y --downloadonly
createrepo /usr/local/repos
ansible mysql -m shell -a "yum clean all"
ansible mysql -m shell -a "yum makecache"

#上传文件 
[lvs01]cd /usr/local/tools
rz mha4mysql-node-0.56-0.el6.noarch.rpm  mha4mysql-manager-0.56-0.el6.noarch.rpm
ansible mysql -m shell -a "cp /usr/local/tools/mha4mysql-node-0.56-0.el6.noarch.rpm /usr/local/src/"
ansible mysql -m shell -a "rpm -ivh /usr/local/src/mha4mysql-node-0.56-0.el6.noarch.rpm"
```



4、在mysql主库中创建mha需要的用户，在MHA Manager安装MHA Manager

```shell
[mysql01]grant usage privileges on *.* to mha@'192.128.1.%' identified by '123456';

[lvs01]yum install -y perl-Config-Tiny epel-release perl-Log-Dispatch perl-Parallel-ForkManager perl-Time-HiRes --downloadonly
createrepo /usr/local/repos

[mysql03]yum clean all && yum makecache
yum install -y perl-Config-Tiny epel-release perl-Log-Dispatch perl-Parallel-ForkManager perl-Time-HiRes
rpm -ivh /usr/local/src/rpm -ivh mha4mysql-manager-0.56-0.el6.noarch.rpm

#创建配置文件目录
 mkdir -p /etc/mha
#创建日志目录
 mkdir -p /var/log/mha/cluster1
#编辑mha配置文件
vim /etc/mha/cluster1.cnf
[server default]
manager_log=/var/log/mha/cluster1/manager        
manager_workdir=/var/log/mha/cluster1            
master_binlog_dir=/data/mysql/data    
master_ip_failover_script=/usr/local/bin/master_ip_failover
user=mha                                   
password=123456                             
ping_interval=2
repl_password=123456
repl_user=root
ssh_user=root                               
[server1]                                   
hostname=192.168.1.51
port=3306                                  
[server2]            
hostname=192.168.1.52
port=3306
[server3]
hostname=192.168.1.53
port=3306

#编辑vip故障转移脚本
vim /usr/local/bin/master_ip_failover
#!/usr/bin/env perl
#  Copyright (C) 2011 DeNA Co.,Ltd.
#  You should have received a copy of the GNU General Public License
#   along with this program; if not, write to the Free Software
#  Foundation, Inc.,
#  51 Franklin Street, Fifth Floor, Boston, MA  02110-1301  USA

## Note: This is a sample script and is not complete. Modify the script based on your environment.

use strict;
use warnings FATAL => 'all';

use Getopt::Long;
use MHA::DBHelper;

my (
  $command,        $ssh_user,         $orig_master_host,
  $orig_master_ip, $orig_master_port, $new_master_host,
  $new_master_ip,  $new_master_port,  $new_master_user,
  $new_master_password
);

my $vip = '192.168.1.102/24';
my $key = '1';
my $ssh_start_vip = "ip addr add $vip eth0:$key";
my $ssh_stop_vip = "ip addr del $vip eth0:$key";
GetOptions(
  'command=s'             => \$command,
  'ssh_user=s'            => \$ssh_user,
  'orig_master_host=s'    => \$orig_master_host,
  'orig_master_ip=s'      => \$orig_master_ip,
  'orig_master_port=i'    => \$orig_master_port,
  'new_master_host=s'     => \$new_master_host,
  'new_master_ip=s'       => \$new_master_ip,
  'new_master_port=i'     => \$new_master_port,
  'new_master_user=s'     => \$new_master_user,
  'new_master_password=s' => \$new_master_password,
);

exit &main();

sub main {
    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";
    if ( $command eq "stop" || $command eq "stopssh" ) {
        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {
        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}
sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
     return 0  unless  ($ssh_user);
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}
sub usage {
  print
"Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}



[mysql01]ip addr add 192.168.1.102/24 dev eth0:1

[mysql03]
#互信检查
masterha_check_ssh  --conf=/etc/mha/cluster1.cnf 
#备份检查
masterha_check_repl  --conf=/etc/mha/cluster1.cnf 
#启动MHA
nohup masterha_manager --conf=/etc/mha/cluster1.cnf --remove_dead_master_conf --ignore_last_failover  < /dev/null> /var/log/mha/cluster1/manager.log 2>&1 &
#状态检查，如果成功会返回OK
masterha_check_status --conf=/etc/mha/cluster1.cnf

#关闭mha的命令
masterha_stop --conf=/etc/mha/cluster1.cnf

#测试,如果能成功登录mysql，说明vip起作用
[mysql02] mysql -uroot -h 192.168.1.102 -p
```



三、搭建mycat   [mycat01和mycat02]

```shell

1、配置JDK环境
[lvs01]cd /usr/local/tools
rz  jdk-8u131-linux-x64.tar.gz Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz
ansible mycat -m shell -a "cp /usr/local/tools/jdk-8u131-linux-x64.tar.gz /usr/local/src/"
ansible mycat -m shell -a "cp /usr/local/tools/Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz /usr/local/src/"


[mycat]

cd /usr/local/src
tar -zxf jdk-8u131-linux-x64.tar.gz

vim /etc/profile
export JAVA_HOME=/usr/local/src/jdk1.8.0_131
export CLASSPATH=.:${JAVA_HOME}/jre/lib/rt.jar:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
export PATH=$PATH:${JAVA_HOME}/bin
source /etc/profile

2、安装并配置mycat
tar -zxf Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz
mv ./mycat /usr/local/
cd /usr/local/mycat
vim ./conf/server.xml
<user name="root">
                <property name="password">123456</property>
                <property name="schemas">TESTDB</property>
...
<user name="admin">
                <property name="password">123456</property>   #只需要改这个密码，但不改也没影响
                <property name="schemas">TESTDB</property>
                <property name="readOnly">true</property>


vim ./conf/schema.xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
      <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1"></schema>               <dataNode name="dn1" dataHost="localhost1" database="db1" />
      <dataHost name="localhost1" maxCon="1000" minCon="10" balance="1"
           writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
             <heartbeat>select user()</heartbeat>
             <!-- can have multi write hosts -->
             <writeHost host="hostM1" url="192.168.1.102:3306" user="root"
                                password="123456">
                     <!-- can have multi read hosts -->
                     <readHost host="hostS1" url="192.168.1.52:3306" user="root" password="123456" />
                     <readHost host="hostS2" url="192.168.1.53:3306" user="root" password="123456" />
                </writeHost>
        </dataHost>
</mycat:schema>


#启动mycat
/usr/local/mycat/bin/mycat start 
ss -pntul | grep 066   #出现9066和8066就说明mycat启动成功

[mysql01]mysql -uroot -h 192.168.1.41 -P 8066 -p  #登录成功看到TESTDB数据库就说明mycat已经成功对mysql进行读写分离。

```

四、配置keepalived使mycat实现高可用

[lvs] vim /etc/keepalived/keepalived.conf

```shell
vrrp_instance VI_2 {
    state BACKUP     #lvs02  设置为Master
    interface eth0
    virtual_router_id 52
    priority 90      #lvs02  设置为100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.1.103/24  brd 192.168.1.255 dev eth0 label eth0:1
    }

}

virtual_server 192.168.1.103 8066 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 2
    protocol TCP

real_server 192.168.1.41 8066 {

    TCP_CHECK {
        connect_timeout 3
        nb_get_retry 3
        delay_before_retry 3
        connect_port 8066
    }
}

real_server 192.168.1.42 8066 {

    TCP_CHECK {
        connect_timeout 3
        nb_get_retry 3
        delay_before_retry 3
        connect_port 8066
    }

  }

}
           
[lvs]sysetmctl restart keepalived

#mycat两个服务器也要lo网卡绑定VIP地址并修改内核参数抑制ARP响应
#华为云主机也要设置虚拟ip绑定到两个mycat服务器
[mycat]
ip addr add 192.168.1.103/32 dev lo

cat >>/etc/sysctl.conf<<EOF
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
EOF
sysctl -p


[mysql]mysql -uroot -h 192.168.1.103 -P 8066 -p   能成功登录说明keepalived的高可用也设置成功！

#工作量比较大，回顾一下，我们就是先做mysql主从复制——>mha高可用——>mycat读写分离——>lvs+keepalived保证mycat高可用！
```

​     
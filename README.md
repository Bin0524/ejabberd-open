

# Startalk EJABBERD

Startalk(前身叫Qtalk，目前主体app尚未全部改名完毕。)是基于ejabberd，根据业务需要改造而来。修改和扩展了很多
ejaberd不支持的功能。



## 关键功能

-   分布式：去掉了依赖mnesia集群的代码，来支持更大的集群，以及防止由于网络分区导致的集群状态不一致。
-   消息处理：通过ejabberd和kafka相连接，实现了消息的路由和订阅发布，可以对消息添加更丰富的处理逻辑。
-   &#x2026;

## Startalk 模块

### Startalk 主要包含：

+ [ejabberd](https://github.com/qunarcorp/ejabberd-open)

IM核心组件，负责维持与客户端的长连接和消息路由

+ [or](https://github.com/qunarcorp/or_open)

IM负载均衡组件，负责验证客户端身份，以及转发http请求到对应的后台服务
+ [im_http_service](https://github.com/qunarcorp/im_http_service_open)

IM HTTP接口服务，负责IM相关数据的查询、设置以及历史消息同步(基于tomcat的java服务)

+ [qfproxy](https://github.com/qunarcorp/qfproxy_open)

IM文件服务，负责文件的上传和下载(基于tomcat的java服务)

+ [push_service](https://github.com/qunarcorp/push_service_open)

IM的push服务，用于离线消息的推送(基于tomcat的java服务)

+ [qtalk_serach](https://github.com/qunarcorp/qtalk_search)

提供远程搜索人员和群的服务

+ redis

IM缓存服务

+ postgresql

IM数据库服务

### Startalk 各个模块之间的关系

![architecture](image/arch.png)

## 注意事项
以下文档请务必仔细阅读和参照执行，否则可能会创建失败

强调几点：
* 切勿使用root账号进行如下操作，很多软件会检查当前用户名称，so请新建用户来进行操作;
* redis 启动需要加载配置
* 对startalk来说，配置中的domain 非常重要，请务必仔细配置，保持一致。
* 在开始之前请务必保证以下几个端口没有被占用：
```
openresty服务：8080
im_http_service服务：8005 8009 8081
qfproxy服务：8006 8010 8082
push_service服务：8007 8011 8083
qtalk_search服务：8888

im服务： 5202 10050

db: 5432 

redis: 6379
```

## 安装

前提条件(如果主机名，用户名和这里的不一致，则需要将安装步骤中的换成自己的名字)：

+ 服务器要求：centos7.x
+ 主机名是：startalk.com
+ hosts添加： 127.0.0.1 startalk.com(sudo vim /etc/hosts)
+ 所有项目都安装到/startalk下面
+ 安装用户和用户组是：startalk:startalk，要保证startalk用户有sudo权限
+ 家目录下有download文件夹，所有文件会下载到该文件夹下
+ 数据库用户名密码是ejabberd:123456，服务地址是：127.0.0.1
+ redis密码是：123456，服务地址是：127.0.0.1
+ 数据库初始化sql在doc目录下
+ 保证可访问主机的：5202、8080端口（关掉防火墙：sudo systemctl stop firewalld.service）
+ IM服务的域名是:qtalk(大家安装线上之前，最好确定好这个值，一旦定了，之后修改的成本就很高，可以参考[domain修改](https://github.com/qunarcorp/ejabberd-open/wiki/host%E4%BF%AE%E6%94%B9)来修改)
+ tls证书：默认安装用的是一个测试证书，线上使用，请更换/startalk/ejabberd/etc/ejabberd/server.pem文件，生成方法见[securing-ejabberd-with-tls-encryption](https://blog.process-one.net/securing-ejabberd-with-tls-encryption/)
+ 出现文件覆盖提示时，输入yes敲回车即可
+ 安装文档中#开头输入的命令表示root执行的，$开头的命令表示普通用户

### 依赖包
```
# yum -y install epel-release
# yum -y update
# yum -y groupinstall Base "Development Tools" "Perl Support"
# yum -y install openssl openssl-devel unixODBC unixODBC-devel pkgconfig libSM libSM-devel libxslt ncurses-devel libyaml libyaml-devel expat expat-devel libxml2-devel libxml2 java-1.8.0-openjdk  java-1.8.0-openjdk-devel  pam-devel pcre-devel gd-devel bzip2-devel zlib-devel libicu-devel libwebp-devel gmp-devel curl-devel postgresql-devel libtidy libtidy-devel recode aspell libmcrypt  libmemcached gd readline-devel libxslt-devel vim
```

### 添加host

```
# vim /etc/hosts
添加下面一行
127.0.0.1 startalk.com
```

### 新建安装用户

```
# groupadd startalk
# useradd -g startalk startalk
# passwd startalk

# groupadd postgres
# useradd -g postgres postgres
# passwd postgres  
```

### 新建安装目录

```
# mkdir /startalk
# chown startalk:startalk /startalk
```

### 为startalk用户添加sudo权限

```
# visudo 

在行
root    ALL= (ALL)    ALL
行后添加

startalk     ALL= (ALL)    ALL
postgres     ALL= (ALL)    ALL

保存后退出
```

### 下载源码

```
# su - startalk
$ mkdir /startalk/download
$ cd /startalk/download
$ git clone https://github.com/qunarcorp/ejabberd-open.git
$ git clone https://github.com/qunarcorp/or_open.git
$ git clone https://github.com/qunarcorp/qtalk_search.git

$ cp ejabberd-open/doc/qtalk.sql /startalk/
$ cp ejabberd-open/doc/init.sql /startalk/
$ chmod 777 /startalk/qtalk.sql
```

### 检测端口使用：

```
# sudo netstat -antlp | egrep "8080|8005|8009|8081|8006|8010|8082|8007|8011|8083|8888|10056|5202|10050|5280|6379"
若没有任何输出，怎表明没有程序占用startalk使用的端口，否则需要关闭已经在使用端口的程序
```

### redis安装

```
$ sudo yum install -y redis
$ sudo vim /etc/redis.conf
将对应的配置修改为下面内容 
daemonize yes
requirepass 123456
maxmemory 134217728
 
启动redis
$ sudo redis-server /etc/redis.conf
 
确认启动成功：
$ sudo netstat -antlp | grep 6379
tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN      8813/redis-server 1
```

### 数据库安装

```
１ 下载源代码
$ wget https://ftp.postgresql.org/pub/source/v11.1/postgresql-11.1.tar.gz
 
2 编译安装
#解压
$ tar -zxvf postgresql-11.1.tar.gz
$ cd postgresql-11.1/
$ sudo ./configure --prefix=/opt/pg11 --with-perl --with-libxml --with-libxslt
 
$ sudo make world
#编译的结果最后必须如下，否则需要检查哪里有error
#PostgreSQL, contrib, and documentation successfully made. Ready to install.
 
$ sudo make install-world
#安装的结果做后必须如下，否则没有安装成功
#PostgreSQL installation complete.
 
3. 添加postgres OS用户
$ sudo mkdir -p /export/pg110_data
  
$ sudo chown postgres:postgres /export/pg110_data
 
4. 创建数据库实例
$ su - postgres
 
$ /opt/pg11/bin/initdb -D /export/pg110_data
 
5. 修改数据库配置文件

将/export/pg110_data/postgresql.conf中的logging_collector的值改为on

6. 启动DB实例
 
$ /opt/pg11/bin/pg_ctl -D /export/pg110_data start
确认启动成功
$ sudo netstat -antlp | grep 5432
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      4751/postmaster     
 
7. 初始化DB结构
 
$ /opt/pg11/bin/psql -U postgres -d postgres -f /startalk/qtalk.sql
$ /opt/pg11/bin/psql -U postgres -d ejabberd -f /startalk/init.sql
 
8. 初始化DB user: ejabberd的密码
 
$ /opt/pg11/bin/psql -U postgres -d postgres -c "ALTER USER ejabberd WITH PASSWORD '123456';"
 
9 psql连接数据库

$ psql -U postgres -d ejabberd -h 127.0.0.1
psql (9.2.24, server 11.1)
WARNING: psql version 9.2, server version 11.0.
         Some psql features might not work.
Type "help" for help.

ejabberd=# select * from host_users;
```

### openresty安装

```
$ su - startalk
$ cd /startalk/download
$ wget https://openresty.org/download/openresty-1.13.6.2.tar.gz
$ tar -zxvf openresty-1.13.6.2.tar.gz
$ cd openresty-1.13.6.2
$ ./configure --prefix=/startalk/openresty --with-http_auth_request_module
$ make
$ make install

or安装
$ cd /startalk/download/or_open
$ cp -rf conf /startalk/openresty/nginx
$ cp -rf lua_app /startalk/openresty/nginx

or操作
启动：/startalk/openresty/nginx/sbin/nginx

确认启动成功
$ sudo netstat -antlp | grep 8080
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      23438/nginx: master
```

### 安装erlang

```
$ cd /startalk/download
$ wget http://erlang.org/download/otp_src_19.3.tar.gz
$ tar -zxvf otp_src_19.3.tar.gz
$ cd otp_src_19.3
$ ./configure --prefix=/startalk/erlang1903
$ make
$ make install

添加PATH
$ vim ~/.bash_profile
 
----------------------------------
# User specific environment and startup programs
ERLANGPATH=/startalk/erlang1903
PATH=$PATH:$HOME/bin:$ERLANGPATH/bin
----------------------------------
 
$ . ~/.bash_profile

确认erlang安装成功
$ erl
Erlang/OTP 19 [erts-8.3] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false]

Eshell V8.3  (abort with ^G)
1> 
```

### 安装ejabberd

```
$ cd /startalk/download
$ cd ejabberd-open/
$ ./configure --prefix=/startalk/ejabberd --with-erlang=/startalk/erlang1903 --enable-pgsql --enable-full-xml
$ make
$ make install
$ cp ejabberd.yml.qunar /startalk/ejabberd/etc/ejabberd/ejabberd.yml
$ cp ejabberdctl.cfg.qunar /startalk/ejabberd/etc/ejabberd/ejabberdctl.cfg

启动ejabberd

$ cd /startalk/ejabberd
启动
$ ./sbin/ejabberdctl start

确认ejabberd安装成功
$ ps -ef | grep 's ejabberd'
startalk 23515     1  4 09:58 ?        00:00:03 /startalk/erlang1903/lib/erlang/erts-8.3/bin/beam.smp -K true -P 250000 -- -root /startalk/erlang1903/lib/erlang -progname erl -- -home /home/startalk -- -name ejabberd@startalk.com -noshell -noinput -noshell -noinput -mnesia dir "/startalk/ejabberd/var/lib/ejabberd" -ejabberd log_rate_limit 20000 log_rotate_size 504857600 log_rotate_count 41 log_rotate_date "$D0" -s ejabberd -smp auto start
```

### 安装java服务(/startalk/download/or_open/deps/tomcat/下的是打好包的三个java服务，自己也可以使用源码打包，然后自己部署)

```
$ cd /startalk/download/
$ cp -rf or_open/deps/tomcat /startalk/
$ cd /startalk/tomcat

修改导航地址和扩展键盘：
$  vim /startalk/tomcat/im_http_service/webapps/im_http_service/WEB-INF/classes/nav.json
$  vim /startalk/tomcat/im_http_service/webapps/im_http_service/WEB-INF/classes/androidqtalk.json
$  vim /startalk/tomcat/im_http_service/webapps/im_http_service/WEB-INF/classes/androidstartalk.json
$  vim /startalk/tomcat/im_http_service/webapps/im_http_service/WEB-INF/classes/iosqtalk.json
$  vim /startalk/tomcat/im_http_service/webapps/im_http_service/WEB-INF/classes/iosstartalk.json

将ip替换成对应机器的ip地址

修改推送服务的地址

$ vim /startalk/tomcat/push_service/webapps/push_service/WEB-INF/classes/app.properties
#使用星语push url
qtalk_push_url=http://ip:8091/qtapi/token/sendPush.qunar
#使用星语push key
qtalk_push_key=12342a14-e6c0-463f-90a0-92b8faec4063

启动java服务
$ cd /startalk/tomcat/im_http_service
$ ./bin/startup.sh


$ cd /startalk/tomcat/qfproxy
$ ./bin/startup.sh

$ cd /startalk/tomcat/push_service
$ ./bin/startup.sh

确认服务启动成功
$ sudo netstat -antlp | egrep '8081|8082|8083|8009|8010|8011|8005|8006|8007'
tcp6       0      0 127.0.0.1:8007          :::*                    LISTEN      23853/java          
tcp6       0      0 :::8009                 :::*                    LISTEN      23748/java          
tcp6       0      0 :::8010                 :::*                    LISTEN      23785/java          
tcp6       0      0 :::8011                 :::*                    LISTEN      23853/java          
tcp6       0      0 :::8081                 :::*                    LISTEN      23748/java          
tcp6       0      0 :::8082                 :::*                    LISTEN      23785/java          
tcp6       0      0 :::8083                 :::*                    LISTEN      23853/java          
tcp6       0      0 127.0.0.1:8005          :::*                    LISTEN      23748/java          
tcp6       0      0 127.0.0.1:8006          :::*                    LISTEN      23785/java 
```

### 安装后端搜索服务
```
安装python3 (3以上都可以，以3.6为标准)
$ cd /startalk/download/qtalk_search
$ sudo yum install https://centos7.iuscommunity.org/ius-release.rpm
$ sudo yum install python36u
安装pip3
$ sudo yum -y install python-pip
所需模块见/startalk/download/qtalk_search/requirements.txt, 建议使用virtualenv部署模块所需环境 (如不使用将系统级安装python3.6, 容易引起和大多数centos自带python2.7的冲突, 同时也需要自行安装python3的pip, 并指定pip3安装所需模块):
$ sudo pip install -U virtualenv （安装virtualenv）
$ sudo pip install --upgrade pip
$ virtualenv --system-site-packages -p python3.6 ./venv （在当前目录下创建venv环境）
启动环境
$ source venv/bin/activate
配置conf/configure.ini, 具体参数详见文件内注释, 如无特殊需求可不修改
$ sudo vim ./conf/configure.ini
安装项目所需模块(如未安装virtualenv, 需sudo yum install python36u-pip, 并使用sudo pip3.6代替命令中默认的pip)
$ pip install -r requirements.txt
设置PYTHONPATH
$ export PYTHONPATH=path/to/project/qtalk_search:$PYTHONPATH
后台启动
$ supervisord -c conf/supervisor.conf
$ supervisorctl -c conf/supervisor.conf reload
确保服务启动（观察日志,确保无报错）
$ tail -100f log/access.log 
```
可以执行以下脚本来检查一些常见的错误
[check.sh](https://github.com/qunarcorp/or_open/blob/master/tools/check.sh)
```
# sed -i 's/ip/自己的ip/g' ./check.sh
# chmod +x check.sh
# ./check.sh
```

到此，服务端已经安装完成。
请下载[startalk客户端](https://im.qunar.com/new/#/download)

客户端配置导航地址：[http://ip:8080/newapi/nck/qtalk_nav.qunar](http://ip:8080/newapi/nck/qtalk_nav.qunar)，使用账号：admin，密码：testpassword登陆(将ip替换成自己服务器的ip)

客户端配置导航的说明[配置导航](https://im.qunar.com/#/platform/access_guide/config_navs?id=config_navs)

可以在二维码生成网站[http://www.liantu.com/](http://www.liantu.com/)生成导航地址的二维码，然后通过扫码在手机客户端添加导航

请参考wiki和接口文档进行相关了解

## 分支和PR

当前，我们项目的分支是：

* master(主干分支)
* release(用于合并到master的准发布分支)
* develop(当前开发分支)
* v1.0(1.0版本的分支)
* v1.1(1.1版本的分支，当前最新稳定分支)
* v2.0(2.0分支版本，支持Erlang/OTP 21.2)


大家提交pull request的时候，可以根据不同分支的功能，合并到不同的分支

## wiki地址

[wiki页](https://github.com/qunarcorp/ejabberd-open/wiki)
## 配置文件修改

参考文档[setting.md](doc/setting.md)

## 接口文档

参考文档[interface.md](doc/interface.md)

## 开发指南

- [developer guide](https://docs.ejabberd.im/developer/guide/)

## 问题反馈

- qchat@qunar.com（邮件）
- 852987381（QQ群）
- QQ群二维码 ![qq](image/qq.png)

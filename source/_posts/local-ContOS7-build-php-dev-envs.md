---
title: build yourself Linux dev env 
date: 2019-05-09 12:17:23
categories: docker nginx
tags: 
- php
---

## `ContOS7`  php多版本环境的配置

### 使用最小化安装之后的第一个问题，内外网不通

- `vi /etc/sysconfig/network-scripts/ifcfg-ens33`
- 修改 `ONBOOT=yes` 后 `systemctl restart network.service`
- 重启主机，如果此时内外网通了但是 `yum list` 失败，主要是服务不可达，考虑为 `DNS` 的原因
- 修改 `vi /etc/resolv.conf` 修改 `dns` 为 `nameserver 8.8.8.8`
  
  ```shell
   yum update # 成功
   yum install wget -y # 装一个下载工具
   mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup # 备份
   wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo # 添加阿里源
   yum clean all # 清除缓存
   yum makecache # 生成缓存
   yum update -y # 更新
  
  ```

### 一些准备工作

- `yum install vim -y && yum install net-tools -y`   习惯了 `vim` 加上一个网络分析小工具
- `yum install gcc gcc-c++ httpd-tools autoconf pcre pcre-devel make automake -y` 安装一些编译测试工具
- `systemctl stop firewalld.service && systemctl disable firewalld.service` 关闭并禁用 防火墙
- `vim /etc/selinux/config`
- 修改 `SELINUX=disabled` 千万不要改错位置  
- 下面的 `selinuxtype` 要是改成 `disable` 建议删了重装会快一些
- 不然 `selinux` 可能会导致你的网络端口不可访问

### 安装 `nginx`

- [参考](http://nginx.org/en/linux_packages.html#RHEL-CentOS)
- `vim /etc/yum.repos.d/nginx.repo` 创建 `yum` 源文件
- 根据官方提示修改为
  
  ```conf
  [nginx]
  name=nginx repo
  baseurl=http://nginx.org/packages/centos/7/$basearch/
  gpgcheck=0
  enabled=1
  ```

- `yum list|grep nginx` 能够看到 `nginx`
  
  ```shell
    [root@localhost ~]# yum list|grep nginx
    nginx.x86_64                                1:1.16.0-1.el7.ngx         nginx
    nginx-debug.x86_64                          1:1.8.0-1.el7.ngx          nginx
    nginx-debuginfo.x86_64                      1:1.16.0-1.el7.ngx         nginx
    nginx-module-geoip.x86_64                   1:1.16.0-1.el7.ngx         nginx
    nginx-module-geoip-debuginfo.x86_64         1:1.16.0-1.el7.ngx         nginx
    nginx-module-image-filter.x86_64            1:1.16.0-1.el7.ngx         nginx
    nginx-module-image-filter-debuginfo.x86_64  1:1.16.0-1.el7.ngx         nginx
    nginx-module-njs.x86_64                     1:1.16.0.0.3.1-1.el7.ngx   nginx
    nginx-module-njs-debuginfo.x86_64           1:1.16.0.0.3.1-1.el7.ngx   nginx
    nginx-module-perl.x86_64                    1:1.16.0-1.el7.ngx         nginx
    nginx-module-perl-debuginfo.x86_64          1:1.16.0-1.el7.ngx         nginx
    nginx-module-xslt.x86_64                    1:1.16.0-1.el7.ngx         nginx
    nginx-module-xslt-debuginfo.x86_64          1:1.16.0-1.el7.ngx         nginx
    nginx-nr-agent.noarch                       2.0.0-12.el7.ngx           nginx
    pcp-pmda-nginx.x86_64                       4.1.0-5.el7_6              updates  
  ```

- `yum install nginx -y` 安装 `nginx`
- `systemctl enable nginx` 开机自启 `nginx`
- `systemctl start nginx` 启动 `nginx`
- 然后输入 `IP` 到浏览器访问成功
- 默认配置位置 `/etc/nginx/nginx.conf` `/etc/nginx/conf.d/default.conf`

### 安装 `PHP`

- [参考](https://webtatic.com/packages/php72/)

  ```shell
  yum install epel-release
  rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
  ```

- 安装 `php` 全扩展 `yum install mod_php72w php72w-bcmath php72w-cli php72w-common php72w-dba php72w-devel php72w-embedded php72w-enchant php72w-fpm php72w-gd  php72w-imap php72w-interbase php72w-imap php72w-intl php72w-ldap php72w-mbstring php72w-mysqlnd php72w-odbc php72w-opcache php72w-pdo php72w-pdo_dblib php72w-pear php72w-pecl-apcu php72w-pecl-imagick php72w-pecl-mongodb php72w-pgsql php72w-phpdbg php72w-process php72w-pspell php72w-recode php72w-snmp php72w-soap php72w-sodium php72w-tidy php72w-xml php72w-xmlrpc -y`
- 配置文件位置 `/etc/php.ini`
- `php -m` 就可以查看装了哪些扩展 嗯 基本都开了
- `systemctl start php-fpm` 虽然是一台机器但我们还是用 `php-fpm` 模式,便于后期加入 `docker` 的扩展 也可以使用 `.sock` 的模块加载模式 理论上会快一些
- `systemctl enable php-fpm` 开机自启

- 发现一个坑，并没有 `redis` 扩展 原生进行编译扩展
  - `wget http://pecl.php.net/get/redis-4.2.0.tgz`
  - `tar zxvf redis-4.2.0.tgz`
  - `cd redis-4.2.0`
  - `/usr/bin/phpize`(这个根据 `phpize` 实际情况来)
  - `./configure --with-php-config=/usr/bin/php-config`(这个根据 `php-config` 实际情况来)
  - `make && make install`
  - `vim /etc/php.d/redis.ini` 这个根据实际情况去决定 是改 `php.ini` 还是别的什么
  - 写入 `extension=redis.so`
  - `systemctl restart php-fpm` 就 `ok` 了

### `php`与`nginx` 链接起来

  ```shell
  [root@localhost ~]# cd dev
  [root@localhost dev]# mkdir -p www/php/test/
  [root@localhost dev]# cd www/php/test/
  [root@localhost test]# echo "<?php phpinfo();" >> ./index.php
  ```

- `vim /etc/nginx/conf.d/default.conf`
  
  ```shell
  location / {
        root   /dev/www/php/test;
        index index.php  index.html index.htm;
    }
  
    location ~* \.php$ {
        fastcgi_index   index.php;
        fastcgi_pass    127.0.0.1:9000;
        include         fastcgi_params;
        fastcgi_param SCRIPT_FILENAME /dev/www/php/test$fastcgi_script_name;
    }
  ```

- `nginx -t && nginx -s reload`

### 安装docker

```shell
yum remove docker  docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine

yum install -y yum-utils

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install docker-ce docker-ce-cli containerd.io -y

systemctl start docker

systemctl enable docker
```

### 使用`docker`构建 `php5.4` 环境

- 更换镜像，使用阿里云的镜像加速
- `https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors`
- `vim /etc/docker/daemon.json`
- 修改成自己的镜像加速
- `"registry-mirrors": ["https://xxxxx.mirror.aliyuncs.com"]`
- `systemctl daemon-reload`
- `systemctl restart docker`
- 运行生成一个`php5.4` 的容器

  ```shell
  docker run -d \
  -v /dev/www/php/:/var/www/html \
  -p 9001:9000 \
  --name phpfpm54 --privileged=true php:5.4-fpm
  ```

#### docker 中 扩展安装

- `docker exec -ti phpfpm54 /bin/bash`
- `-ti` 打开图形界面
- `/bin/bash` 执行 这个命令
- `exec` 可以在 不 `exit` 的情况下进行附加操作
- 进去后需要 `apt-get update` 里面是 `debian9` 系统
- `docker-php-ext-install pdo_mysql` 来装 `pdo_mysql` 扩展
- 以下这些扩展都可以安装 不过好多有坑0.0

```shell
bcmath bz2 calendar ctype curl dba dom enchant exif fileinfo filter ftp gd gettext gmp hash iconv imap interbase intl json ldap mbstring mysqli oci8 odbc opcache pcntl pdo pdo_dblib pdo_firebird pdo_mysql pdo_oci pdo_odbc pdo_pgsql pdo_sqlite pgsql phar posix pspell readline recode reflection session shmop simplexml snmp soap sockets sodium spl standard sysvmsg sysvsem sysvshm tidy tokenizer wddx xml xmlreader xmlrpc xmlwriter xsl zend_test zip
```

#### docker `Debian` 换源安装`gd` 库

```shell
cd /etc/apt && cp sources.list ./sources.list.bak
apt-get install vim

vim sources.list
deb http://mirrors.163.com/debian/ stretch main non-free contrib
deb http://mirrors.163.com/debian/ stretch-updates main non-free contrib
deb http://mirrors.163.com/debian/ stretch-backports main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch-updates main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch-backports main non-free contrib

//如果连apt-get update都做不到   可以先 rm sources.list 再创一个新的
echo "deb http://mirrors.163.com/debian/ stretch main non-free contrib
deb http://mirrors.163.com/debian/ stretch-updates main non-free contrib
deb http://mirrors.163.com/debian/ stretch-backports main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch-updates main non-free contrib
deb-src http://mirrors.163.com/debian/ stretch-backports main non-free contrib" >> sources.list
//将源更换为上述  新的源

apt-get update
apt-get install libfreetype6-dev -y
apt-get install libpng-dev -y
apt-get install libjpeg62-turbo-dev -y
//如果源没出问题 上述应该都能安装
docker-php-ext-configure gd --with-freetype-dir=/usr/include --with-jpeg-dir=/usr/include
//先编译gd库的扩展
docker-php-ext-install gd
// 这样就在docker容器内完美安装了gd库
```

- 装 `redis` 扩展

```shell
curl -L -o /tmp/redis.tar.gz https://github.com/phpredis/phpredis/archive/3.1.3.tar.gz

tar xfz /tmp/redis.tar.gz

rm -r /tmp/redis.tar.gz

mkdir -p /usr/src/php/ext

mv phpredis-3.1.3 /usr/src/php/ext/redis

docker-php-ext-install redis
```

- 退出测试
- `vim /etc/nginx/conf.d/default.conf`
- 修改 `fastcgi_param SCRIPT_FILENAME /var/www/html/test$fastcgi_script_name;`
- 这里由于是在容器内执行所以修改路径为这个
- 设置 `phpstrom` 的 `sftp` 自动上传到 本地虚拟机

- `ps aux|grep php`

```shell
1 root 0:00 php-fpm: master process (/usr/local/etc/php-fpm.conf)
7 www-data 0:00 php-fpm: pool www
8 www-data 0:00 php-fpm: pool www

# 发现id 1就是php了，给它个信号让它重启即可，执行命令如下：
kill -USR2 1
```

- 如果是在外面也可以：docker exec -it 容器id或名称 kill -USR2 1
- 或者重启容器  docker restart 容器id或名称

#### 安装  `swoole`

- `yum install git openssl-devel -y`
- `git clone https://gitee.com/swoole/swoole.git`
- 高版本
  - `cd swoole`
  - `git checkout v4.4.6`
  - `./configure --enable-openssl --enable-mysqlnd --enable-sockets`
  - `make clean && make && sudo make install`
  - `cd /etc/php.d/`
  - `cp sockets.ini ./swoole.ini`
  - `vim swoole.ini 修改为 swoole`
  - `systemctl restart php-fpm`
- 低版本
  - `wget https://github.com/redis/hiredis/archive/v0.14.0.zip`
  - `yun install -y unzip`
  - `unzip v0.14.0.zip`
  - `cd hiredis-0.14.0`
  - `make -j`

    ```shell
    ./configure \
    --with-php-config=/usr/bin/php-config \
    --enable-openssl  \
    --enable-http2  \
    --enable-async-redis \
    --enable-sockets \
    --enable-mysqlnd && make clean && make && sudo make install
    ```

    ```shell
    cd /etc/php.d/
    cp sockets.ini ./swoole.ini
    ls
    vim swoole.ini 修改为 swoole
    systemctl restart php-fpm
    ```

```shell

[root@localhost swoole]# php --ri swoole

swoole

Swoole => enabled
Author => Swoole Team <team@swoole.com>
Version => 4.4.0-alpha
Built => May 10 2019 20:30:06
coroutine => enabled
epoll => enabled
eventfd => enabled
signalfd => enabled
cpu_affinity => enabled
spinlock => enabled
rwlock => enabled
sockets => enabled
openssl => OpenSSL 1.0.2k-fips  26 Jan 2017
http2 => enabled
pcre => enabled
zlib => enabled
mutex_timedlock => enabled
pthread_barrier => enabled
futex => enabled
mysqlnd => enabled
async_redis => enabled

Directive => Local Value => Master Value
swoole.enable_coroutine => On => On
swoole.display_errors => On => On
swoole.use_shortname => On => On
swoole.unixsock_buffer_size => 8388608 => 8388608

```

### docker 构建生产可用 php8.1

```dockerfile
FROM php:8.1-fpm

ENV REDIS_VERSION=5.3.5  SWOOLE_VERSION=4.8.6

# 设置时间
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo 'Asia/Shanghai' > /etc/timezone

#更换源
RUN sed -i "s/deb.debian.org/mirrors.aliyun.com/g" /etc/apt/sources.list
RUN sed -i "s/http/https/g" /etc/apt/sources.list

#编译安装扩展 gd 与其他依赖配置
RUN apt-get update &&\
    apt-get install --allow-downgrades -y  libz-dev unzip sudo  \
       libzip-dev libfreetype6-dev libjpeg62-turbo-dev libpng-dev libssl-dev libcurl4-openssl-dev &&\
    apt-get clean &&\
    apt-get autoremove &&\
    docker-php-ext-configure gd &&\
    docker-php-ext-install -j$(nproc) gd

# top 与 ps 这类的监控软件 ，需要时添加
#RUN apt install procps --allow-downgrades -y

#pecl 安装扩展 redis
RUN pecl install redis-$REDIS_VERSION && docker-php-ext-enable redis

#安装其他必需扩展
RUN docker-php-ext-install bcmath pdo_mysql zip sockets pcntl

#安装swoole
#RUN pecl install swoole-$SWOOLE_VERSION && docker-php-ext-enable swoole

# Composer安装 与设置镜像地址
RUN curl -sS https://getcomposer.org/installer | php \
    && mv composer.phar /usr/local/bin/composer \
    && composer self-update --clean-backups \
    && composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/

# php-OPcache 配置
RUN touch /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini &&  \
    echo "[opcache]" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "zend_extension=opcache.so" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.enable=1" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.enable_cli=1" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.memory_consumption=256" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.interned_strings_buffer=8" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.max_accelerated_files=100000" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.max_wasted_percentage=5" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.use_cwd=1" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.validate_timestamps=1" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.revalidate_freq=2" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.save_comments=0" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.jit_debug=0" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.jit=1255" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "opcache.jit_buffer_size=256M" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "# 其他php配置" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "memory_limit=256M" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "upload_max_filesize=50M" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
    echo "post_max_size=50M" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini

# 更改 fpm 的分组 因为我 宿主机有 www:x:1001:1001 分组跟用户 所以才做这样的处理 其他机器按照自己想要的分组去处理就好
# groupadd -g 1001 www
RUN  useradd -u 1001 www && RUN sed -i "s/www-data/www/g" /usr/local/etc/php-fpm.d/www.conf

```

### 构建镜像

`docker build -t php81-laravel:v1 .`

### 构建运行时

```shell
docker run -i -t -d \
 --restart=always \
 -p 8100:9000 \
 -v /pathto/code:/var/www/html  --name=laravel8  php81-laravel:v1
```

### 执行者权限问题

- 在容器内 `php-fpm` 的运行时分组 为 `www-data`

```shell
# 容器内
id  www-data 
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

- 所以其生成的 log 文件也是 属于这个分组下
- 在宿主机上 如果没有 33 的分组与用户， 展示的则为 `tape tape`
- 如若在宿主机上使用 crontab 势必会导致 log 权限的问题，
- 解决方式 就是保证内外用户uid 的一致 如:

```dockerfile
RUN  useradd -u 1001 www && RUN sed -i "s/www-data/www/g" /usr/local/etc/php-fpm.d/www.conf
```

### laravel 相关任务

```shell
# 可以直接在宿主机上 执行 容器内的命令 如
# 将容器内的 composer 进行更新
docker exec composer update -vvv 
#（此时以容器内的 root 用户去执行的命令）

# 所以在容器内不安装 crontab 的情况下，可以在宿主机进行配置
# 以 root 用户去启动 crontab  在容器内转换成 www 用户去执行 php 完美
crontab -e

* * * * *  docker exec laravel8 sudo -u www php artisan schedule:run
```



## lnmp 一键安装 脚本

- 支持到 php8.1 , 测试服用这个省心
- `https://lnmp.org/`

## `Debian` 环境配置

因为 `contOS8` 以后的版本会面临的风险，所以打算更换Linux的构建版本，在对比了一系列版本后，最终选择了 `Debian11` 希望这个系统能陪我10-20年。

### ISO 文件获取

- 11版本的的debian.iso 文件获取 发行版本也才300M
[https://developer.aliyun.com/mirror/debian-cd](https://developer.aliyun.com/mirror/debian-cd)

### VirtualBox 相关配置

- 内存调整 4G
- 核心 两核
- 网卡开 nat 跟 host only 两个

- 安装过程中拒绝 `ghome` 等桌面软件, 开启 `ssh serve`

### 初始化后调整本地的host网络

```shell

# 查看相关网卡
ip addr 
# 发现 enp0s8 host only 网卡没配置

# 对其进行配置
vim /etc/network/interfaces

allow-hotplug enp0s8
iface enp0s8 inet dhcp

# 通过 ifup 来启动网卡
ifup enp0s8
```

### ssh 开启允许 root 登录

```shell
vim /etc/ssh/sshd_config

# 修改
PermitRootLogin yes

# 重启 
systemctl restart sshd

```

- [更改apt源](https://developer.aliyun.com/mirror/debian)

```shell
# 更改源
vim /etc/apt/sources.list
# 写入数据
deb http://mirrors.aliyun.com/debian/ bullseye main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ bullseye main non-free contrib
deb http://mirrors.aliyun.com/debian-security/ bullseye-security main
deb-src http://mirrors.aliyun.com/debian-security/ bullseye-security main
deb http://mirrors.aliyun.com/debian/ bullseye-updates main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ bullseye-updates main non-free contrib
deb http://mirrors.aliyun.com/debian/ bullseye-backports main non-free contrib
deb-src http://mirrors.aliyun.com/debian/ bullseye-backports main non-free contrib

# 更新源与 相关软件
apt update && apt upgrade -y

```

### docker 与相关应用安装

- docker [安装](https://docs.docker.com/engine/install/debian/)

```shell

apt install ca-certificates curl gnupg lsb-release -y

curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg


echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" |  tee /etc/apt/sources.list.d/docker.list > /dev/null

# 更新源
apt update

# 直接安装
apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y 

# 查看版本 & 进行安装
apt-cache madison docker-ce

apt-get install docker-ce=5:20.10.16~3-0~debian-bullseye docker-ce-cli=5:20.10.16~3-0~debian-bullseye containerd.io docker-compose-plugin

# 持久化开启docker
systemctl start docker && systemctl enable docker

# 镜像加速
vim /etc/docker/daemon.json
# 写入数据
{
  "registry-mirrors": ["https://xxxx.mirror.aliyuncs.com"]
}

# 重启加速
systemctl daemon-reload && systemctl restart docker

```

### nginx 安装

```shell
apt install curl gnupg2 ca-certificates lsb-release debian-archive-keyring -y

apt update  && apt install nginx -y

# 持久化开启
systemctl start nginx && systemctl enable nginx
```

### 个性化设置

```shell

# ll 别名
alias ll="ls -l"

# oh-my-zsh 安装

# 安装 zsh
apt-get install zsh -y

# 切换shell 到zsh   然后重启
chsh -s /bin/zsh

# https://github.com/ohmyzsh/ohmyzsh 想办法下载安装脚本
sh install.sh

# 更换主题
vim ~/.zshrc

# 修改 ZSH_THEME="cloud" 新开tab生效

# 更多主题 可以查看 https://github.com/ohmyzsh/ohmyzsh/wiki/themes

# xshell连接此host的终端类型，改成linux 来避免 home end 失效
```

## `docker` 安装基础服务

### `docker` 安装 `portainer`

```shell
# 拉取镜像
docker pull portainer/portainer
# 生成数据保存路径
mkdir -p /data/portainer_data
# 生成 portainer 运行时
docker run -d -p 9001:9000 --name portainer --restart=always \
-v /var/run/docker.sock:/var/run/docker.sock  -v  /data/portainer_data:/data  portainer/portainer 
```

### `docker`  安装 `mysql`服务

- ``
- `` #用于挂载`mysql`数据文件
- `` #用于挂载`mysql`配置文件
  
```shell
# 安装MySQL5.7
dokcer pull mysql:5.7
# 生成mysql数据保存路径
mkdir -p /data/mysql/datadir
# 生成mysql配置文件保存路径
mkdir -p /data/mysql/conf.d

# 生成 MySQL 运行时
docker run -d \
--name mysql5.7 -p 3306:3306 \
-v /data/mysql/datadir:/var/lib/mysql \
-v /data/mysql/conf.d:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=123456  mysql:5.7

# `-e`：设置环境变量，此处指定 `root` 密码
```

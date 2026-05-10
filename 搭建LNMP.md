```
vi /etc/sysconfig/network-scripts/ifcfg-ens33 
#修改ONBOOT的值为yes
#:wq!   保存并推出
# 重启网络服务
systemctl restart network
```

查看IP信息方便后面的远程连接

```
ip addr
# 输出：
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:e5:c0:45 brd ff:ff:ff:ff:ff:ff
    inet 192.168.227.175/24 brd 192.168.227.255 scope global noprefixroute dynamic ens33
       valid_lft 3595sec preferred_lft 3595sec
    inet6 fe80::d5af:7a9a:3615:2477/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

由于创建虚拟机的时候没有创建用户，但日常操作最好使用普通用户，进公司上班99%是普通用户root用户容易出问题

```
#  创建普通用户 wangcan（-m 自动创建家目录）
useradd -m wangcan

#  为 wangcan 用户设置密码
passwd wangcan
# 输出：
Changing password for user wangcan.
New password:  # 输入密码（如未满足复杂度要求会报错）
Retype new password: 
passwd: all authentication tokens updated successfully.  # 密码设置成功提示
```

切换到wangcan用户

```
su wangcan
```

尝试使用sudo修改主机名

```
sudo hostnamectl set-hostname master
# 输出：
We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:
    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.
[sudo] password for wangcan:  # 输入 wangcan 密码
wangcan is not in the sudoers file.  This incident will be reported.  # 权限不足报错
```

<font color="red">出现问题,权限不足</font>

root用户创建的wangcan普通用户尚未被加入到系统的特权组，普通用户想要使用sudo 就需要将用户加入允许使用sudo的组中,先回到root用户再执行下面的命令

```
usermod -aG wheel wangcan
```

解决权限问题之后切换到wangcan用户重新使用命令修改主机名

```
[root@localhost ~]# su wangcan
[wangcan@localhost root]$ cd
[wangcan@localhost ~]$ sudo hostnamectl set-hostname master
[wangcan@localhost ~]$ 
```

修改成功后就需要安装nginx了

但是CentOS官方源里默认不包含Nginx需要先安装EPEL库它能提供大量企业级软件

```
[wangcan@master ~]# sudo yum install -y epel-release
已加载插件：fastestmirror
Determining fastest mirrors
Could not retrieve mirrorlist http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=os&infra=stock error was
14: curl#6 - "Could not resolve host: mirrorlist.centos.org; 未知的错误"


 One of the configured repositories failed (未知),
 and yum doesn't have enough cached data to continue. At this point the only
 safe thing yum can do is fail. There are a few ways to work "fix" this:

     1. Contact the upstream for the repository and get them to fix the problem.
......
     5. Configure the failing repository to be skipped, if it is unavailable.
        Note that yum will try to contact the repo. when it runs most commands,
        so will have to try and fail each time (and thus. yum will be be much
        slower). If it is a very temporary problem though, this is often a nice
        compromise:

            yum-config-manager --save --setopt=<repoid>.skip_if_unavailable=true

Cannot find a valid baseurl for repo: base/7/x86_64
[root@master ~]# 
```

<font color="red">Could not resolve host: mirrorlist.centos.org; 未知的错误"    经典的错误，**意思是服务器无法解析域名mirrorlist.centos.org,所以yum找不到，不知道从哪里下载软件**</font>

```
[wangcan@master ~]# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=107 time=194 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=107 time=189 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=107 time=235 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=107 time=251 ms
64 bytes from 8.8.8.8: icmp_seq=7 ttl=107 time=1221 ms
64 bytes from 8.8.8.8: icmp_seq=8 ttl=107 time=221 ms
64 bytes from 8.8.8.8: icmp_seq=9 ttl=107 time=283 ms
64 bytes from 8.8.8.8: icmp_seq=10 ttl=107 time=272 ms
^C
--- 8.8.8.8 ping statistics ---
10 packets transmitted, 8 received, 20% packet loss, time 9009ms
rtt min/avg/max/mdev = 189.777/358.928/1221.994/327.709 ms, pipe 2
[wangcan@master ~]# ping www.baidu.com
PING www.a.shifen.com (183.2.172.177) 56(84) bytes of data.
64 bytes from 183.2.172.177 (183.2.172.177): icmp_seq=1 ttl=52 time=128 ms
64 bytes from 183.2.172.177 (183.2.172.177): icmp_seq=2 ttl=52 time=110 ms
64 bytes from 183.2.172.177 (183.2.172.177): icmp_seq=3 ttl=52 time=43.0 ms
64 bytes from 183.2.172.177 (183.2.172.177): icmp_seq=5 ttl=52 time=1064 ms
64 bytes from 183.2.172.177 (183.2.172.177): icmp_seq=6 ttl=52 time=63.9 ms
64 bytes from 183.2.172.177 (183.2.172.177): icmp_seq=7 ttl=52 time=57.6 ms
^C
--- www.a.shifen.com ping statistics ---
7 packets transmitted, 6 received, 14% packet loss, time 6010ms
rtt min/avg/max/mdev = 43.009/244.719/1064.504/367.851 ms, pipe 2
wangcan@master ~]# 
```

上面两个命令可以看出虚拟机的网络配置完全没问题，DNS也能正常工作

执行下面的命令实现一键替换源地址

```
[wangcan@master ~]$ sudo cp -r /etc/yum.repos.d /etc/yum.repos.d.backup  #先备份
[sudo] wangcan 的密码：

# 将 CentOS 官方源地址从 mirror.centos.org 改为 vault.centos.org（归档站）
[wangcan@master ~]$ sudo sed -i 's/mirror.centos.org/vault.centos.org/g' /etc/yum.repos.d/CentOS-*.repo
[sudo] wangcan 的密码：
[wangcan@master ~]$ sudo sed -i 's/#baseurl/baseurl/g' /etc/yum.repos.d/CentOS-*.repo
[wangcan@master ~]$ sudo sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*.repo
```

<font color="green">CentOS 7 在 **2024 年 6 月 30 日正式停止维护（EOL）**，官方已经把 `mirror.centos.org` 上的所有 CentOS 7 软件包下架了现在访问`mirror.centos.org/centos/7/` 会直接返回 404 或 301 跳转这个地址已经`vault.centos.org` 是官方「历史归档站」，100% 兼容 CentOS 7,不会出现「国内镜像同步不全、包缺失」的问题，是**唯一官方兜底方案**，能保证 yum 安装时不会因为包找不到而报错。</font>

清理缓存

```
[wangcan@master ~]$ sudo yum clean all
已加载插件：fastestmirror
正在清理软件源： base extras updates
Cleaning up list of fastest mirrors
```

重建缓存，让新地址生效

```
[wangcan@master ~]$ sudo yum makecache
已加载插件：fastestmirror
Determining fastest mirrors
base                                                                                                           | 3.6 kB  00:00:00     
extras                                                                                                         | 2.9 kB  00:00:00     
updates                                                                                                        | 2.9 kB  00:00:00     
(1/10): base/7/x86_64/group_gz                                                                                 | 153 kB  00:00:03     
(2/10): base/7/x86_64/filelists_db                                                                             | 7.2 MB  00:00:11     
(3/10): base/7/x86_64/other_db                                                                                 | 2.6 MB  00:00:03     
(4/10): base/7/x86_64/primary_db                                                                               | 6.1 MB  00:00:16     
(5/10): extras/7/x86_64/primary_db                                                                             | 253 kB  00:00:14     
(6/10): extras/7/x86_64/filelists_db                                                                           | 305 kB  00:00:15     
(7/10): extras/7/x86_64/other_db                                                                               | 154 kB  00:00:01     
(8/10): updates/7/x86_64/filelists_db                                                                          |  15 MB  00:03:04     
(9/10): updates/7/x86_64/other_db                                                                              | 1.6 MB  00:00:28     
updates/7/x86_64/primary_db    FAILED                                          ===============-     ]   14 B/s |  53 MB 157:14:19 ETA 
http://vault.centos.org/centos/7/updates/x86_64/repodata/f19044932626155f0cd849e88972b84875fc85e3308b4d622844a911c4ef54d0-primary.sqlite.bz2: [Errno 12] Timeout on https://vault.centos.org/centos/7/updates/x86_64/repodata/f19044932626155f0cd849e88972b84875fc85e3308b4d622844a911c4ef54d0-primary.sqlite.bz2: (28, 'Operation too slow. Less than 1000 bytes/sec transferred the last 30 seconds')
正在尝试其它镜像。
updates/7/x86_64/primary_db    FAILED                                          
http://vault.centos.org/centos/7/updates/x86_64/repodata/f19044932626155f0cd849e88972b84875fc85e3308b4d622844a911c4ef54d0-primary.sqlite.bz2: [Errno 14] HTTPS Error 301 - Moved Permanently
正在尝试其它镜像。
updates/7/x86_64/primary_db                                                                                    |  27 MB  00:00:08     
元数据缓存已建立
[wangcan@master ~]$ 
```

<font color="green">先安装EPEL，EPEL里面有很多高质量的第三方库，**先装 EPEL，相当于给你的 CentOS 开了 “软件库扩展权限”，后面装很多工具都不用再折腾了**。</font>EPEL 是地基，让你能拿到各种工具；Nginx 官方源是建材，让你能拿到最优质的特定材料。

```
[wangcan@master ~]$ sudo yum install -y epel-release
[sudo] wangcan 的密码：
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
正在解决依赖关系
--> 正在检查事务
---> 软件包 epel-release.noarch.0.7-11 将被 安装
--> 解决依赖关系完成

依赖关系解决

======================================================================================================================================
 Package                             架构                          版本                           源                             大小
======================================================================================================================================
正在安装:
 epel-release                        noarch                        7-11                           extras                         15 k

事务概要
======================================================================================================================================
安装  1 软件包

总下载量：15 k
安装大小：24 k
Downloading packages:
警告：/var/cache/yum/x86_64/7/extras/packages/epel-release-7-11.noarch.rpm: 头V3 RSA/SHA256 Signature, 密钥 ID f4a80eb5: NOKEY:-- ETA 
epel-release-7-11.noarch.rpm 的公钥尚未安装
epel-release-7-11.noarch.rpm                                                                                   |  15 kB  00:00:02     
从 file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7 检索密钥
导入 GPG key 0xF4A80EB5:
 用户ID     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 指纹       : 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 软件包     : centos-release-7-9.2009.0.el7.centos.x86_64 (@anaconda)
 来自       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在安装    : epel-release-7-11.noarch                                                                                          1/1 
  验证中      : epel-release-7-11.noarch                                                                                          1/1 

已安装:
  epel-release.noarch 0:7-11                                                                                                          

完毕！
[wangcan@master ~]$ 
```

# 安装nginx

```
[wangcan@master ~]$ sudo yum install -y nginx
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
epel/x86_64/metalink                                                                                           | 3.5 kB  00:00:00     
 * epel: ftp-stud.hs-esslingen.de
epel                                                                                                           | 4.3 kB  00:00:00     
(1/3): epel/x86_64/group                                                                                       | 399 kB  00:00:16     
(2/3): epel/x86_64/updateinfo                                                                                  | 1.0 MB  00:00:35     
epel/x86_64/primary_db         FAILED                                                               ]  6.8 B/s | 3.7 MB 273:21:23 ETA 
http://mirror.math.princeton.edu/pub/fedora-archive/epel/7/x86_64/repodata/8b9b4a93a490a3787b09f182250cbaba737fc9c51bc560beecf33988d1abce01-primary.sqlite.gz: [Errno 12] Timeout on http://mirror.math.princeton.edu/pub/fedora-archive/epel/7/x86_64/repodata/8b9b4a93a490a3787b09f182250cbaba737fc9c51bc560beecf33988d1abce01-primary.sqlite.gz: (28, 'Operation too slow. Less than 1000 bytes/sec transferred the last 30 seconds')
正在尝试其它镜像。
(3/3): epel/x86_64/primary_db                                                                                  | 8.7 MB  00:04:57     
正在解决依赖关系
--> 正在检查事务
---> 软件包 nginx.x86_64.1.1.20.1-10.el7 将被 安装
--> 正在处理依赖关系 nginx-filesystem = 1:1.20.1-10.el7，它被软件包 1:nginx-1.20.1-10.el7.x86_64 需要
--> 正在处理依赖关系 libcrypto.so.1.1(OPENSSL_1_1_0)(64bit)，它被软件包 1:nginx-1.20.1-10.el7.x86_64 需要
--> 正在处理依赖关系 libssl.so.1.1(OPENSSL_1_1_0)(64bit)，它被软件包 1:nginx-1.20.1-10.el7.x86_64 需要
--> 正在处理依赖关系 libssl.so.1.1(OPENSSL_1_1_1)(64bit)，它被软件包 1:nginx-1.20.1-10.el7.x86_64 需要
--> 正在处理依赖关系 nginx-filesystem，它被软件包 1:nginx-1.20.1-10.el7.x86_64 需要
--> 正在处理依赖关系 redhat-indexhtml，它被软件包 1:nginx-1.20.1-10.el7.x86_64 需要
--> 正在处理依赖关系 libcrypto.so.1.1()(64bit)，它被软件包 1:nginx-1.20.1-10.el7.x86_64 需要
--> 正在处理依赖关系 libprofiler.so.0()(64bit)，它被软件包 1:nginx-1.20.1-10.el7.x86_64 需要
--> 正在处理依赖关系 libssl.so.1.1()(64bit)，它被软件包 1:nginx-1.20.1-10.el7.x86_64 需要
--> 正在检查事务
---> 软件包 centos-indexhtml.noarch.0.7-9.el7.centos 将被 安装
---> 软件包 gperftools-libs.x86_64.0.2.6.1-1.el7 将被 安装
---> 软件包 nginx-filesystem.noarch.1.1.20.1-10.el7 将被 安装
---> 软件包 openssl11-libs.x86_64.1.1.1.1k-7.el7 将被 安装
--> 解决依赖关系完成

依赖关系解决

======================================================================================================================================
 Package                              架构                       版本                                  源                        大小
======================================================================================================================================
正在安装:
 nginx                                x86_64                     1:1.20.1-10.el7                       epel                     588 k
为依赖而安装:
 centos-indexhtml                     noarch                     7-9.el7.centos                        base                      92 k
 gperftools-libs                      x86_64                     2.6.1-1.el7                           base                     272 k
 nginx-filesystem                     noarch                     1:1.20.1-10.el7                       epel                      24 k
 openssl11-libs                       x86_64                     1:1.1.1k-7.el7                        epel                     1.5 M

事务概要
======================================================================================================================================
安装  1 软件包 (+4 依赖软件包)

总下载量：2.4 M
安装大小：6.7 M
Downloading packages:
nginx-filesystem-1.20.1-10.el7 FAILED                                                               ]  0.0 B/s |  32 kB  --:--:-- ETA 
https://mirror.flo.c-f.ro/fedora/archive/epel/7/x86_64/Packages/n/nginx-filesystem-1.20.1-10.el7.noarch.rpm: [Errno 14] HTTPS Error 404 - Not Found
正在尝试其它镜像。
To address this issue please refer to the below wiki article 

https://wiki.centos.org/yum-errors

If above article doesn't help to resolve this issue please use https://bugs.centos.org/.

(1/5): centos-indexhtml-7-9.el7.centos.noarch.rpm                                                              |  92 kB  00:00:03     
(2/5): gperftools-libs-2.6.1-1.el7.x86_64.rpm                                                                  | 272 kB  00:00:04     
warning: /var/cache/yum/x86_64/7/epel/packages/nginx-1.20.1-10.el7.x86_64.rpm: Header V4 RSA/SHA256 Signature, key ID 352c64e5: NOKEY 
nginx-1.20.1-10.el7.x86_64.rpm 的公钥尚未安装
(3/5): nginx-1.20.1-10.el7.x86_64.rpm                                                                          | 588 kB  00:00:05     
(4/5): nginx-filesystem-1.20.1-10.el7.noarch.rpm                                                               |  24 kB  00:00:05     
(5/5): openssl11-libs-1.1.1k-7.el7.x86_64.rpm                                                                  | 1.5 MB  00:01:49     
--------------------------------------------------------------------------------------------------------------------------------------
总计                                                                                                   23 kB/s | 2.4 MB  00:01:49     
从 file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 检索密钥
导入 GPG key 0x352C64E5:
 用户ID     : "Fedora EPEL (7) <epel@fedoraproject.org>"
 指纹       : 91e9 7d7c 4a5e 96f1 7f3e 888f 6a2f aea2 352c 64e5
 软件包     : epel-release-7-11.noarch (@extras)
 来自       : /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在安装    : 1:openssl11-libs-1.1.1k-7.el7.x86_64                                                                              1/5 
  正在安装    : 1:nginx-filesystem-1.20.1-10.el7.noarch                                                                           2/5 
  正在安装    : centos-indexhtml-7-9.el7.centos.noarch                                                                            3/5 
  正在安装    : gperftools-libs-2.6.1-1.el7.x86_64                                                                                4/5 
  正在安装    : 1:nginx-1.20.1-10.el7.x86_64                                                                                      5/5 
  验证中      : gperftools-libs-2.6.1-1.el7.x86_64                                                                                1/5 
  验证中      : centos-indexhtml-7-9.el7.centos.noarch                                                                            2/5 
  验证中      : 1:nginx-filesystem-1.20.1-10.el7.noarch                                                                           3/5 
  验证中      : 1:nginx-1.20.1-10.el7.x86_64                                                                                      4/5 
  验证中      : 1:openssl11-libs-1.1.1k-7.el7.x86_64                                                                              5/5 

已安装:
  nginx.x86_64 1:1.20.1-10.el7                                                                                                        

作为依赖被安装:
  centos-indexhtml.noarch 0:7-9.el7.centos      gperftools-libs.x86_64 0:2.6.1-1.el7      nginx-filesystem.noarch 1:1.20.1-10.el7     
  openssl11-libs.x86_64 1:1.1.1k-7.el7         

完毕！
[wangcan@master ~]$ 
```

启动nginx并查看状态

```
[wangcan@master ~]$ sudo systemctl start nginx
[sudo] wangcan 的密码：
[wangcan@master ~]$ sudo systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since 六 2026-04-25 21:58:46 CST; 9s ago
  Process: 111836 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 111832 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 111830 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 111838 (nginx)
   CGroup: /system.slice/nginx.service
           ├─111838 nginx: master process /usr/sbin/nginx
           ├─111839 nginx: worker process
           └─111840 nginx: worker process

4月 25 21:58:46 master systemd[1]: Starting The nginx HTTP and reverse proxy server...
4月 25 21:58:46 master nginx[111832]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
4月 25 21:58:46 master nginx[111832]: nginx: configuration file /etc/nginx/nginx.conf test is successful
4月 25 21:58:46 master systemd[1]: Started The nginx HTTP and reverse proxy server.
[wangcan@master ~]$ 
```

在Windows 浏览器里输入虚拟机 IP `192.168.227.175`（或你当前的 `ens33` IP），回车

![image-20260425150327191](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260425150327191.png)

并没有出现欢迎页面

突然想起我并没有打开防火墙80端口

```
[wangcan@master ~]$ sudo firewall-cmd --permanent --add-port=80/tcp
[sudo] wangcan 的密码：
success
[wangcan@master ~]$ sudo firewall-cmd --reload   #重启防火墙
success
[wangcan@master ~]$ 
```

重新在Windows 浏览器里输入虚拟机 IP `192.168.227.175`（或你当前的 `ens33` IP），回车

![image-20260425151120605](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260425151120605.png)

在我没有询问AI时我以为这表明nginx服务安装成功了

但在我询问AI后才知道图片表明Apache(httpd)的默认欢迎页，不是Nginx的！

<a id="anchor-1"></a>说明服务器上现在同时跑了两个Web服务，80端口被Apache占用了，Nginx没有工作

原因：CentOS7系统默认自带了httpd（Apache)服务，它也会占用80端口访问 IP 看到的这个 CentOS 欢迎页，就是 Apache 的默认首页，不是 Nginx 的

解决方法：

```
#停止并禁用 Apache
sudo systemctl stop httpd
sudo systemctl disable httpd
```

```
#重启 Nginx
sudo systemctl restart nginx
```

重新去浏览器搜索IP

![image-20260426105121628](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260426105121628.png)

看到欢迎界面，服务安装成功

---

# 安装MySQL

因为CentOS7默认源是MariaDB但需要的是MySQL

所以需要下载一个MySQL官方仓库安装包

```
[wangcan@master ~]$ sudo yum install -y https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
[sudo] wangcan 的密码：
已加载插件：fastestmirror
mysql80-community-release-el7-3.noarch.rpm                                                                     |  25 kB  00:00:00     
正在检查 /var/tmp/yum-root-x8HkI9/mysql80-community-release-el7-3.noarch.rpm: mysql80-community-release-el7-3.noarch
/var/tmp/yum-root-x8HkI9/mysql80-community-release-el7-3.noarch.rpm 将被安装
正在解决依赖关系
--> 正在检查事务
---> 软件包 mysql80-community-release.noarch.0.el7-3 将被 安装
--> 解决依赖关系完成

依赖关系解决

======================================================================================================================================
 Package                               架构               版本              源                                                   大小
======================================================================================================================================
正在安装:
 mysql80-community-release             noarch             el7-3             /mysql80-community-release-el7-3.noarch              31 k

事务概要
======================================================================================================================================
安装  1 软件包

总计：31 k
安装大小：31 k
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在安装    : mysql80-community-release-el7-3.noarch                                                                            1/1 
  验证中      : mysql80-community-release-el7-3.noarch                                                                            1/1 

已安装:
  mysql80-community-release.noarch 0:el7-3                                                                                            

完毕！
[wangcan@master ~]$ 
```

然后安装MySQL服务器

```
[wangcan@master ~]$ sudo yum install -y mysql-community-server
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * epel: ftp-stud.hs-esslingen.de
正在解决依赖关系
--> 正在检查事务
---> 软件包 mysql-community-server.x86_64.0.8.0.46-1.el7 将被 安装
--> 正在处理依赖关系 mysql-community-common(x86-64) = 8.0.46-1.el7，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
--> 正在处理依赖关系 mysql-community-icu-data-files = 8.0.46-1.el7，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
--> 正在处理依赖关系 mysql-community-client(x86-64) >= 8.0.11，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
--> 正在处理依赖关系 /usr/bin/perl，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
--> 正在处理依赖关系 libaio.so.1(LIBAIO_0.1)(64bit)，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
--> 正在处理依赖关系 libaio.so.1(LIBAIO_0.4)(64bit)，它被软件包 mysql-community-server-8.0.46-
......
(31/36): perl-podlators-2.5.1-3.el7.noarch.rpm                                                                 | 112 kB  00:00:00     
(32/36): perl-macros-5.16.3-299.el7_9.x86_64.rpm                                                               |  44 kB  00:00:02     
(33/36): perl-threads-shared-1.43-6.el7.x86_64.rpm                                                             |  39 kB  00:00:00     
(34/36): perl-threads-1.87-4.el7.x86_64.rpm                                                                    |  49 kB  00:00:00     
(35/36): perl-libs-5.16.3-299.el7_9.x86_64.rpm                                                                 | 690 kB  00:00:04     
(36/36): mysql-community-server-8.0.46-1.el7.x86_64.rpm                                                        |  65 MB  00:00:48     
--------------------------------------------------------------------------------------------------------------------------------------
总计                                                                                                  1.5 MB/s | 101 MB  00:01:08     
从 file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql 检索密钥
导入 GPG key 0x5072E1F5:
 用户ID     : "MySQL Release Engineering <mysql-build@oss.oracle.com>"
 指纹       : a4a9 4068 76fc bd3c 4567 70c8 8c71 8d3b 5072 e1f5
 软件包     : mysql80-community-release-el7-3.noarch (@/mysql80-community-release-el7-3.noarch)
 来自       : /etc/pki/rpm-gpg/RPM-GPG-KEY-mysql


mysql-community-common-8.0.46-1.el7.x86_64.rpm 的公钥尚未安装


 失败的软件包是：mysql-community-common-8.0.46-1.el7.x86_64
 GPG  密钥配置为：file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

[wangcan@master ~]$ 
```

<font color="red">出现错误  公钥尚未安装, 失败的软件包是：mysql-community-common</font>

<font color="red">Linux 安装软件会**校验密钥**，确保软件没被篡改。</font>

<font color="red">MySQL 8.0.46 用的是**新密钥**，但你系统里还是老密钥，所以报错</font>

先导入新的密钥，再重新安装MySQL

```
[wangcan@master ~]$ sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
[wangcan@master ~]$ sudo yum install -y mysql-community-server
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * epel: ftp-stud.hs-esslingen.de
正在解决依赖关系
--> 正在检查事务
---> 软件包 mysql-community-server.x86_64.0.8.0.46-1.el7 将被 安装
--> 正在处理依赖关系 mysql-community-common(x86-64) = 8.0.46-1.el7，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
--> 正在处理依赖关系 mysql-community-icu-data-files = 8.0.46-1.el7，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
--> 正在处理依赖关系 mysql-community-client(x86-64) >= 8.0.11，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
--> 正在处理依赖关系 /usr/bin/perl，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
--> 正在处理依赖关系 libaio.so.1(LIBAIO_0.1)(64bit)，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
--> 正在处理依赖关系 libaio.so.1(LIBAIO_0.4)(64bit)，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
--> 正在处理依赖关系 net-tools，它被软件包 mysql-community-server-8.0.46-1.el7.x86_64 需要
......
  验证中      : perl-File-Path-2.09-2.el7.noarch                                                                                35/37 
  验证中      : 4:perl-libs-5.16.3-299.el7_9.x86_64                                                                             36/37 
  验证中      : 1:mariadb-libs-5.5.68-1.el7.x86_64                                                                              37/37 

已安装:
  mysql-community-libs.x86_64 0:8.0.46-1.el7                      mysql-community-libs-compat.x86_64 0:8.0.46-1.el7                   
  mysql-community-server.x86_64 0:8.0.46-1.el7                   

作为依赖被安装:
  libaio.x86_64 0:0.3.109-13.el7                                        mysql-community-client.x86_64 0:8.0.46-1.el7                 
  mysql-community-client-plugins.x86_64 0:8.0.46-1.el7                  mysql-community-common.x86_64 0:8.0.46-1.el7                 
  mysql-community-icu-data-files.x86_64 0:8.0.46-1.el7                  net-tools.x86_64 0:2.0-0.25.20131004git.el7                                         
......
4:5.16.3-299.el7_9                          
  perl-macros.x86_64 4:5.16.3-299.el7_9                                 perl-parent.noarch 1:0.225-244.el7                           
  perl-podlators.noarch 0:2.5.1-3.el7                                   perl-threads.x86_64 0:1.87-4.el7                             
  perl-threads-shared.x86_64 0:1.43-6.el7                              

替代:
  mariadb-libs.x86_64 1:5.5.68-1.el7                                                                                                  

完毕！
[wangcan@master ~]$ 

```

安装完毕后打开MySQL并查看状态，设置开机自启

```
[wangcan@master ~]$ sudo systemctl start mysqld
[sudo] wangcan 的密码：
[wangcan@master ~]$ 
[wangcan@master ~]$ sudo systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since 六 2026-04-25 22:54:34 CST; 34s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 67442 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 67570 (mysqld)
   Status: "Server is operational"
   CGroup: /system.slice/mysqld.service
           └─67570 /usr/sbin/mysqld

4月 25 22:54:30 master systemd[1]: Starting MySQL Server...
4月 25 22:54:34 master systemd[1]: Started MySQL Server.
[wangcan@master ~]$ sudo systemctl enable mysqld
[wangcan@master ~]$ 
```

由于MySQL安装后会产生一个临时的root密码，我们需要找到它设置一个新的密码

```
#在文件中查找临时密码所在的行
[wangcan@master ~]$ sudo grep "temporary password" /var/log/mysqld.log
2026-04-25T14:54:31.645222Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: yo2qtEHv4:7+
[wangcan@master ~]$ 
```

MySQL安全初始化，新密码 <font color="red">Root@123456</font>

```
[wangcan@master ~]$ sudo mysql_secure_installation

Securing the MySQL server deployment.

Enter password for user root: 
Error: Access denied for user 'root'@'localhost' (using password: YES)
[wangcan@master ~]$ sudo mysql_secure_installation

Securing the MySQL server deployment.

Enter password for user root: 

The existing password for the user account root has expired. Please set a new password.

New password: 

Re-enter new password: 
The 'validate_password' component is installed on the server.
The subsequent steps will run with the existing configuration
of the component.
Using existing password for root.

Estimated strength of the password: 100 
Change the password for root ? ((Press y|Y for Yes, any other key for No) : y

New password: 

Re-enter new password: 

Estimated strength of the password: 100 
Do you wish to continue with the password provided?(Press y|Y for Yes, any other key for No) : y
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them. This is intended only for
testing, and to make the installation go a bit smoother.
You should remove them before moving into a production
environment.

Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
Success.


Normally, root should only be allowed to connect from
'localhost'. This ensures that someone cannot guess at
the root password from the network.

Disallow root login remotely? (Press y|Y for Yes, any other key for No) : y
Success.

By default, MySQL comes with a database named 'test' that
anyone can access. This is also intended only for testing,
and should be removed before moving into a production
environment.


Remove test database and access to it? (Press y|Y for Yes, any other key for No) : y
 - Dropping test database...
Success.

 - Removing privileges on test database...
Success.

Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.

Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y
Success.

All done! 
[wangcan@master ~]$ 
```

最后四个y的作用：

✅ 移除匿名用户
✅ 禁止 root 远程登录（安全）
✅ 删除 test 数据库
✅ 刷新权限

登录MySQL

```
[wangcan@master ~]$ mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 15
Server version: 8.0.46 MySQL Community Server - GPL

Copyright (c) 2000, 2026, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

下面

# 安装PHP及核心扩展

```
[root@master ~]# sudo yum install -y php php-fpm php-mysqlnd php-gd php-xml php-mbstring
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * epel: mirror.math.princeton.edu
正在解决依赖关系
--> 正在检查事务
---> 软件包 php.x86_64.0.5.4.16-48.el7 将被 安装
--> 正在处理依赖关系 php-common(x86-64) = 5.4.16-48.el7，它被软件包 php-5.4.16-48.el7.x86_64 需要
......
--> 正在检查事务
---> 软件包 libXau.x86_64.0.1.0.8-2.1.el7 将被 安装
--> 解决依赖关系完成

依赖关系解决

=========================================================================================================
 Package                  架构              版本                                源                  大小
=========================================================================================================
正在安装:
 php                      x86_64            5.4.16-48.el7                       base               1.4 M
 php-fpm                  x86_64            5.4.16-48.el7                       base               1.4 M
 php-gd                   x86_64            5.4.16-48.el7                       base               128 k
 php-mbstring             x86_64            5.4.16-48.el7                       base               506 k
 php-mysqlnd              x86_64            5.4.16-48.el7                       base               174 k
 php-xml                  x86_64            5.4.16-48.el7                       base               126 k
为依赖而安装:
 apr                      x86_64            1.4.8-7.el7                         base               104 k
 apr-util                 x86_64            1.5.2-6.el7_9.1                     updates             92 k
 httpd                    x86_64            2.4.6-99.el7.centos.1               updates            2.7 M
......
 t1lib                    x86_64            5.1.2-14.el7                        base               166 k

事务概要
=========================================================================================================
安装  6 软件包 (+16 依赖软件包)

总下载量：11 M
安装大小：39 M
Downloading packages:
(1/22): apr-util-1.5.2-6.el7_9.1.x86_64.rpm                                       |  92 kB  00:00:07     
......  
(22/22): httpd-2.4.6-99.el7.centos.1.x86_64.rpm                                   | 2.7 MB  00:01:24     
---------------------------------------------------------------------------------------------------------
总计                                                                     139 kB/s |  11 MB  00:01:24     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在安装    : apr-1.4.8-7.el7.x86_64                                                              1/22 
  正在安装    : apr-util-1.5.2-6.el7_9.1.x86_64                                                     2/22 
......

  验证中      : libzip-0.10.1-8.el7.x86_64                                                         22/22 

已安装:
  php.x86_64 0:5.4.16-48.el7          php-fpm.x86_64 0:5.4.16-48.el7     php-gd.x86_64 0:5.4.16-48.el7 
  php-mbstring.x86_64 0:5.4.16-48.el7 php-mysqlnd.x86_64 0:5.4.16-48.el7 php-xml.x86_64 0:5.4.16-48.el7

作为依赖被安装:
  apr.x86_64 0:1.4.8-7.el7                         apr-util.x86_64 0:1.5.2-6.el7_9.1                     
  httpd.x86_64 0:2.4.6-99.el7.centos.1             httpd-tools.x86_64 0:2.4.6-99.el7.centos.1            
  libX11.x86_64 0:1.6.7-5.el7_9                    libX11-common.noarch 0:1.6.7-5.el7_9                  
  libXau.x86_64 0:1.0.8-2.1.el7                    libXpm.x86_64 0:3.5.12-2.el7_9                        
  libjpeg-turbo.x86_64 0:1.2.90-8.el7              libxcb.x86_64 0:1.13-1.el7                            
  libzip.x86_64 0:0.10.1-8.el7                     mailcap.noarch 0:2.1.41-2.el7                         
  php-cli.x86_64 0:5.4.16-48.el7                   php-common.x86_64 0:5.4.16-48.el7                     
  php-pdo.x86_64 0:5.4.16-48.el7                   t1lib.x86_64 0:5.1.2-14.el7                           

完毕！
[root@master ~]# 
```

PHP-FPM是PHP的独立服务进程，nginx通过它于PHP通信

```
[root@master ~]# sudo systemctl start php-fpm
[root@master ~]# sudo systemctl status php-fpm
● php-fpm.service - The PHP FastCGI Process Manager
   Loaded: loaded (/usr/lib/systemd/system/php-fpm.service; disabled; vendor preset: disabled)
   Active: active (running) since 六 2026-04-25 23:35:28 CST; 11s ago
 Main PID: 111253 (php-fpm)
   Status: "Processes active: 0, idle: 5, Requests: 0, slow: 0, Traffic: 0req/sec"
   CGroup: /system.slice/php-fpm.service
           ├─111253 php-fpm: master process (/etc/php-fpm.conf)
           ├─111254 php-fpm: pool www
           ├─111255 php-fpm: pool www
           ├─111256 php-fpm: pool www
           ├─111257 php-fpm: pool www
           └─111258 php-fpm: pool www

4月 25 23:35:28 master systemd[1]: Starting The PHP FastCGI Process Manager...
4月 25 23:35:28 master systemd[1]: Started The PHP FastCGI Process Manager.
[root@master ~]# sudo systemctl enable php-fpm   #设置开机自启
Created symlink from /etc/systemd/system/multi-user.target.wants/php-fpm.service to /usr/lib/systemd/system/php-fpm.service.
[root@master ~]# 
```

配置Nginx将PHP请求转发给PHP-PFM，意思是修改Nginx的默认站点配置，让它遇到.php文件时，交给PHP-FPM处理

```

[root@master ~]# vi /etc/nginx/nginx.conf
```

下入下面的内容

```
    server {
        listen       80;
        server_name  _;
        root         /usr/share/nginx/html;

        index index.php index.html index.htm;

        location ~ \.php$ {
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include        fastcgi_params;
        }
    }
```

保存退出

检查语法并重载

```
[root@master ~]# sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@master ~]# sudo systemctl reload nginx
```

创建PHP测试文件并验证

```
[root@master ~]# sudo vi /usr/share/nginx/html/info.php
```

写入

```
<?php phpinfo(); ?>
```

保存，然后在浏览器访问 `http://192.168.227.175/info.php`。

![image-20260425165535801](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260425165535801.png)

看到紫色的 PHP 信息页，LNMP 正式跑通了。

```
[root@master ~]# mysql --version
mysql  Ver 8.0.46 for Linux on x86_64 (MySQL Community Server - GPL)
[root@master ~]# php --version
PHP 5.4.16 (cli) (built: Apr  1 2020 04:07:17) 
Copyright (c) 1997-2013 The PHP Group
Zend Engine v2.4.0, Copyright (c) 1998-2013 Zend Technologies
[root@master ~]# nginx -v
nginx version: nginx/1.20.1
[root@master ~]# 
```

php的版本太低了，CentOS 7 默认的 PHP 5.4 版本过旧，不满足 MySQL 8.0 驱动要求，也不符合企业用人需求。

# 升级PHP

先安装Remi仓库

```
[root@master ~]# sudo yum install -y https://rpms.remirepo.net/enterprise/remi-release-7.rpm
已加载插件：fastestmirror
remi-release-7.rpm                                                                |  28 kB  00:00:00     
正在检查 /var/tmp/yum-root-x8HkI9/remi-release-7.rpm: remi-release-7.9-6.el7.remi.noarch
/var/tmp/yum-root-x8HkI9/remi-release-7.rpm 将被安装
正在解决依赖关系
--> 正在检查事务
---> 软件包 remi-release.noarch.0.7.9-6.el7.remi 将被 安装
--> 解决依赖关系完成

依赖关系解决

=========================================================================================================
 Package                 架构              版本                         源                          大小
=========================================================================================================
正在安装:
 remi-release            noarch            7.9-6.el7.remi               /remi-release-7             39 k

事务概要
=========================================================================================================
安装  1 软件包

总计：39 k
安装大小：39 k
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在安装    : remi-release-7.9-6.el7.remi.noarch                                                   1/1 
  验证中      : remi-release-7.9-6.el7.remi.noarch                                                   1/1 

已安装:
  remi-release.noarch 0:7.9-6.el7.remi                                                                   

完毕！
[root@master ~]# 
```

```
[root@master ~]# sudo yum module distable php -y
已加载插件：fastestmirror
没有该命令：module。请使用 /bin/yum --help
#yum module 是 YUM4 的模块化命令，CentOS 7 的旧版 yum 不支持它，所以报了 没有该命令
```

先安装yum-utils ,yum-config-manager会从Remi仓库拉取新版PHP

```
[root@master ~]# sudo yum install -y yum-utils
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
epel/x86_64/metalink                                                              | 3.5 ......

https://wiki.centos.org/yum-errors

If above article doesn't help to resolve this issue please use https://bugs.centos.org/.

remi-safe                                                                         | 3.0 kB  00:00:00     
updates                                                                           | 2.9 kB  00:00:00     
remi-safe/primary_db           FAILED                                          /s | 1.7 MB  01:59:43 ETA 
https://mirrors.tuna.tsinghua.edu.cn/remi/enterprise/7/safe/x86_64/repodata/b831227aa5dc4f09770c5d7998a19caa8fc8c7ec106482671d8dc4b4980d3107-primary.sqlite.bz2: [Errno 12] Timeout on 
正在尝试其它镜像。
remi-safe/primary_db           FAILED                                          
http://remirepo.reloumirrors.net/enterprise/7/safe/x86_64/repodata/b831227aa5dc4f09770c5d7998a19caa8fc8c7ec106482671d8dc4b4980d3107-primary.sqlite.bz2: [Errno 14] curl#6 - "Could not resolve host: remirepo.reloumirrors.net; Unknown error"
正在尝试其它镜像。
remi-safe/primary_db           FAILED                                          /s | 2.2 MB  05:44:43 ETA 
https://mirror.23m.com/remi/enterprise/7/safe/x86_64/repodata/b831227aa5dc4f09770c5d7998a19caa8fc8c7ec106482671d8dc4b4980d3107-primary.sqlite.bz2: [Errno 12] Timeout on https://mirror.23m.com/remi/enterprise/7/safe/x86_64/repodata/b831227aa5dc4f09770c5d7998a19caa8fc8c7ec106482671d8dc4b4980d3107-primary.sqlite.bz2: (28, 'Operation too slow. Less than 1000 bytes/sec transferred the last 30 seconds')
正在尝试其它镜像。
remi-safe/primary_db           FAILED                                          
https://mirror.team-cymru.com/remi/enterprise/7/safe/x86_64/repodata/b831227aa5dc4f09770c5d7998a19caa8fc8c7ec106482671d8dc4b4980d3107-primary.sqlite.bz2: [Errno 14] curl#7 - "Failed to connect to 216.31.2.234: Network is unreachable"
正在尝试其它镜像。
.....                                         
http://mirror.neolabs.kz/remi/enterprise/7/safe/x86_64/repodata/b831227aa5dc4f09770c5d7998a19caa8fc8c7ec106482671d8dc4b4980d3107-primary.sqlite.bz2: [Errno 14] curl#7 - "Failed to connect to 2a00:5da0:1:1::141: Network is unreachable"
正在尝试其它镜像。
remi-safe/primary_db                                                              | 2.6 MB  00:00:04     
正在解决依赖关系
--> 正在检查事务
---> 软件包 yum-utils.noarch.0.1.1.31-54.el7_8 将被 安装
--> 正在处理依赖关系 python-kitchen，它被软件包 yum-utils-1.1.31-54.el7_8.noarch 需要
--> 正在处理依赖关系 libxml2-python，它被软件包 yum-utils-1.1.31-54.el7_8.noarch 需要
--> 正在检查事务
---> 软件包 libxml2-python.x86_64.0.2.9.1-6.el7_9.6 将被 安装
--> 正在处理依赖关系 libxml2 = 2.9.1-6.el7_9.6，它被软件包 libxml2-python-2.9.1-6.el7_9.6.x86_64 需要
---> 软件包 python-kitchen.noarch.0.1.1.1-5.el7 将被 安装
--> 正在处理依赖关系 python-chardet，它被软件包 python-kitchen-1.1.1-5.el7.noarch 需要
--> 正在检查事务
---> 软件包 libxml2.x86_64.0.2.9.1-6.el7.5 将被 升级
---> 软件包 libxml2.x86_64.0.2.9.1-6.el7_9.6 将被 更新
---> 软件包 python-chardet.noarch.0.2.2.1-3.el7 将被 安装
--> 解决依赖关系完成

依赖关系解决

=========================================================================================================
 Package                     架构                版本                         源                    大小
=========================================================================================================
正在安装:
 yum-utils                   noarch              1.1.31-54.el7_8              base                 122 k
为依赖而安装:
 libxml2-python              x86_64              2.9.1-6.el7_9.6              updates              247 k
 python-chardet              noarch              2.2.1-3.el7                  base                 227 k
 python-kitchen              noarch              1.1.1-5.el7                  base                 267 k
为依赖而更新:
 libxml2                     x86_64              2.9.1-6.el7_9.6              updates              668 k

事务概要
=========================================================================================================
安装  1 软件包 (+3 依赖软件包)
升级           ( 1 依赖软件包)

总下载量：1.5 M
Downloading packages:
Delta RPMs disabled because /usr/bin/applydeltarpm not installed.
(1/5): libxml2-python-2.9.1-6.el7_9.6.x86_64.rpm                                  | 247 kB  00:00:07     
(2/5): python-chardet-2.2.1-3.el7.noarch.rpm                                      | 227 kB  00:00:07     
(3/5): python-kitchen-1.1.1-5.el7.noarch.rpm                                      | 267 kB  00:00:08     
(4/5): yum-utils-1.1.31-54.el7_8.noarch.rpm                                       | 122 kB  00:00:01     
(5/5): libxml2-2.9.1-6.el7_9.6.x86_64.rpm                                         | 668 kB  00:00:09     
---------------------------------------------------------------------------------------------------------
总计                                                                     157 kB/s | 1.5 MB  00:00:09     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在更新    : libxml2-2.9.1-6.el7_9.6.x86_64                                                       1/6 
  正在安装    : libxml2-python-2.9.1-6.el7_9.6.x86_64                                       ......                                                    
  验证中      : libxml2-2.9.1-6.el7.5.x86_64                                                         6/6 

已安装:
  yum-utils.noarch 0:1.1.31-54.el7_8                                                                     

作为依赖被安装:
  libxml2-python.x86_64 0:2.9.1-6.el7_9.6               python-chardet.noarch 0:2.2.1-3.el7              
  python-kitchen.noarch 0:1.1.1-5.el7                  

作为依赖被升级:
  libxml2.x86_64 0:2.9.1-6.el7_9.6                                                                       

完毕！
[root@master ~]# 
```

启用仓库，自然覆盖系统旧版本

```
[root@master ~]# sudo yum-config-manager --enable remi-php82
已加载插件：fastestmirror
=========================================== repo: remi-php82 ============================================
[remi-php82]
async = True
bandwidth = 0
base_persistdir = /var/lib/yum/repos/x86_64/7
......
sslclientkey = 
sslverify = True
throttle = 0
timeout = 30.0
ui_id = remi-php82
ui_repoid_vars = releasever,
   basearch
username = 

[root@master ~]# 
```

它会从 Remi 仓库拉取新版 PHP

```
[root@master ~]# sudo yum install -y php php-fpm php-mysqlnd php-gd php-xml php-mbstring
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * epel: mirror.math.princeton.edu
 * remi-php82: mirrors.tuna.tsinghua.edu.cn
 * remi-safe: mirrors.tuna.tsinghua.edu.cn
mysql-connectors-community                                                        | 3.0 kB  00:00:00     
mysql-tools-community                                                             | 3.0 kB  00:00:00     
remi-php82                                                                        | 3.0 kB  00:00:00     
remi-php82/primary_db                                                             | 209 kB  00:00:04     
正在解决依赖关系
--> 正在检查事务
---> 软件包 php.x86_64.0.5.4.16-48.el7 将被 升级
---> 软件包 php.x86_64.0.8.2.20-1.el7.remi 将被 更新
......
---> 软件包 dejavu-fonts-common.noarch.0.2.33-6.el7 将被 安装
---> 软件包 graphite2.x86_64.0.1.3.10-1.el7_3 将被 安装
--> 解决依赖关系完成

依赖关系解决

=========================================================================================================
 Package                          架构            版本                         源                   大小
=========================================================================================================
正在更新:
 php                              x86_64          8.2.20-1.el7.remi            remi-php82          2.0 M
 php-fpm                          x86_64          8.2.20-1.el7.remi            remi-php82          2.1 M
 php-gd                           x86_64          8.2.20-1.el7.remi            remi-php82          102 k
 php-mbstring                     x86_64          8.2.20-1.el7.remi            remi-php82          580 k
 php-mysqlnd                      x86_64          8.2.20-1.el7.remi            remi-php82          255 k
 php-xml                          x86_64          8.2.20-1.el7.remi            remi-php82          252 k
为依赖而安装:
 dejavu-fonts-common              noarch          2.33-6.el7                   base                 64 k
......
 php-pdo                          x86_64          8.2.20-1.el7.remi            remi-php82          158 k

事务概要
=========================================================================================================
安装           ( 15 依赖软件包)
升级  6 软件包 (+ 3 依赖软件包)

总下载量：16 M
Downloading packages:
Delta RPMs disabled because /usr/bin/applydeltarpm not installed.
(1/24): dejavu-fonts-common-2.33-6.el7.noarch.rpm                                 |  64 kB  00:00:04     
......                                    
php-mbstring-8.2.20-1.el7.remi FAILED                                          /s | 8.1 MB  00:02:44 ETA 
http://remi.mirrors.cu.be/enterprise/7/php82/x86_64/php-mbstring-8.2.20-1.el7.remi.x86_64.rpm: [Errno 12] Timeout on http://remi.mirrors.cu.be/enterprise/7/php82/x86_64/php-mbstring-8.2.20-1.el7.remi.x86_64.rpm: (28, 'Connection timed out after 30002 milliseconds')
正在尝试其它镜像。
......    
(24/24): php-cli-8.2.20-1.el7.remi.x8 83% [=======================     ]  11 kB/s |  13 MB  00:04:09 ETA 
......
  验证中      : php-5.4.16-48.el7.x86_64                                                           27/33 
  验证中      : php-mysqlnd-5.4.16-48.el7.x86_64                                                   28/33 
  验证中      : php-cli-5.4.16-48.el7.x86_64                                                       29/33 
  验证中      : php-common-5.4.16-48.el7.x86_64                                                    30/33 
  验证中      : php-fpm-5.4.16-48.el7.x86_64                                                       31/33 
  验证中      : php-pdo-5.4.16-48.el7.x86_64                                                       32/33 
  验证中      : php-gd-5.4.16-48.el7.x86_64                                                        33/33 

作为依赖被安装:
  dejavu-fonts-common.noarch 0:2.33-6.el7           dejavu-sans-fonts.noarch 0:2.33-6.el7                
......          
  php-mysqlnd.x86_64 0:8.2.20-1.el7.remi             php-xml.x86_64 0:8.2.20-1.el7.remi                 

作为依赖被升级:
  php-cli.x86_64 0:8.2.20-1.el7.remi                php-common.x86_64 0:8.2.20-1.el7.remi               
  php-pdo.x86_64 0:8.2.20-1.el7.remi               

完毕！
[root@master ~]# 
```

重新启动并查看状态

```
[root@master ~]# sudo systemctl restart php-fpm
[root@master ~]# sudo systemctl status php-fpm
● php-fpm.service - The PHP FastCGI Process Manager
   Loaded: loaded (/usr/lib/systemd/system/php-fpm.service; enabled; vendor preset: disabled)
   Active: active (running) since 日 2026-04-26 00:45:46 CST; 24s ago
 Main PID: 105447 (php-fpm)
   Status: "Processes active: 0, idle: 5, Requests: 0, slow: 0, Traffic: 0.00req/sec"
   CGroup: /system.slice/php-fpm.service
           ├─105447 php-fpm: master process (/etc/php-fpm.conf)
           ├─105448 php-fpm: pool www
           ├─105449 php-fpm: pool www
           ├─105450 php-fpm: pool www
           ├─105451 php-fpm: pool www
           └─105452 php-fpm: pool www

4月 26 00:45:46 master systemd[1]: Starting The PHP FastCGI Process Manager...
4月 26 00:45:46 master systemd[1]: Started The PHP FastCGI Process Manager.
```

查看版本

```
[root@master ~]# php -version
PHP 8.2.20 (cli) (built: Jun  4 2024 13:22:51) (NTS gcc x86_64)
Copyright (c) The PHP Group
Zend Engine v4.2.20, Copyright (c) Zend Technologies
[root@master ~]# 
```

设置开机自启

```
[root@master ~]# sudo systemctl enable php-fpm
[root@master ~]# sudo systemctl is-enabled php-fpm
enabled
```

去window浏览器查看192.168.227.175/info.php

![image-20260425174945635](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260425174945635.png)

至此PHP升级成功

但是nginx的版本也有点低存在一些安全漏洞等

Nginx升级

先关掉当前的nginx,在备份现有的配置

```
wangcan@master ~]$ sudo systemctl stop nginx
[sudo] wangcan 的密码：
[wangcan@master ~]$  sudo cp -r /etc/nginx /etc/nginx.backup
```

添加Nginx官方稳定版yum库，需要创建一个仓库文件

```
[wangcan@master ~]$ sudo vi /etc/yum.repos.d/nginx.repo
[sudo] wangcan 的密码：
[wangcan@master ~]$ 
```

写入下面的内容（注意复制粘贴的话可能会少几个字母）

```
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
```

安装Nginx新版本

```
[wangcan@master ~]$ sudo yum update nginx
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
epel/x86_64/metalink                                                              | 3.5 kB  00:00:00     
 * epel: fedora-archive.ip-connect.info
 * remi-php82: muug.ca
 * remi-safe: muug.ca   
......
(2/2): pcre2-10.23-2.el7.x86_64.rpm                                               | 201 kB  00:00:04     
---------------------------------------------------------------------------------------------------------
总计                                                                     249 kB/s | 1.0 MB  00:00:04     
从 https://nginx.org/keys/nginx_signing.key 检索密钥
导入 GPG key 0xB49F6B46:
 用户ID     : "nginx signing key <signing-key-2@nginx.com>"
 指纹       : 8540 a6f1 8833 a80e 9c16 53a4 2fd2 1310 b49f 6b46
 来自       : https://nginx.org/keys/nginx_signing.key
是否继续？[y/N]：y
导入 GPG key 0x7BD9BF62:
 用户ID     : "nginx signing key <signing-key@nginx.com>"
 指纹       : 573b fd6b 3d8f bc64 1079 a6ab abf5 bd82 7bd9 bf62
 来自       : https://nginx.org/keys/nginx_signing.key
是否继续？[y/N]：y
导入 GPG key 0x8D88A2B3:
 用户ID     : "nginx signing key <signing-key-3@nginx.com>"
 指纹       : 9e9b e90e acbc de69 fe9b 204c bcdc d8a3 8d88 a2b3
 来自       : https://nginx.org/keys/nginx_signing.key
是否继续？[y/N]：y
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  正在安装    : pcre2-10.23-2.el7.x86_64                                                             1/3 
......
  验证中      : 1:nginx-1.20.1-10.el7.x86_64                                                         3/3 

作为依赖被安装:
  pcre2.x86_64 0:10.23-2.el7                                                                             

更新完毕:
  nginx.x86_64 1:1.26.1-2.el7.ngx                                                                        

完毕！
[root@master ~]# nginx -v
nginx version: nginx/1.26.1
```

合并配置文件，nginx升级后可能会生成一个nginx.conf.rpmnew文件，不必理会，直接使用我们的备份文件替换新版本默认配置即可

```
[root@master ~]# sudo cp /etc/nginx.backup/nginx.conf /etc/nginx/nginx.conf
[root@master ~]# sudo cp -r /etc/nginx.backup/conf.d /etc/nginx
```

测试配置并启动

```
[root@master ~]# sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@master ~]# sudo systemctl start nginx
Job for nginx.service failed. See "systemctl status nginx.service" and "journalctl -xe" for details.
```

发现启动失败出现错误

查看状态

```
[root@master ~]# sudo systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: protocol) since 日 2026-04-26 10:18:15 CST; 1min 19s ago
     Docs: http://nginx.org/en/docs/
  Process: 40625 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)

4月 26 10:18:15 master systemd[1]: Starting nginx - high performance web server...
4月 26 10:18:15 master systemd[1]: Failed to parse PID from file /var/run/nginx.pid: Invalid argument
4月 26 10:18:15 master nginx[40625]: nginx: [emerg] open() "/run/nginx.pid" failed (13: Permissio...ied)
4月 26 10:18:15 master systemd[1]: Failed to start nginx - high performance web server.
4月 26 10:18:15 master systemd[1]: Unit nginx.service entered failed state.
4月 26 10:18:15 master systemd[1]: nginx.service failed.
Hint: Some lines were ellipsized, use -l to show in full.
[root@master ~]# 
```

<font color="red">Failed to parse PID from file /var/run/nginx.pid: Invalid argument</font>

意思是：我们还原的老配置文件里指定的 PID 文件路径（通常是 /var/run/nginx.pid），和升级后 Nginx 1.26.1 版本配合 CentOS 7 的 systemd 所期望的路径不匹配，并且可能遇到了权限问题。

<font color="red">nginx: [emerg] open() "/run/nginx.pid" failed (13: Permissio...ied)</font>

意思是：Nginx：没有权限创建/写入pid文件 导致启动失败

Nginx 启动时需要在 /run/nginx 目录里写自己的进程号文件（nginx.pid），但这个目录丢了 / 权限不对 → 启动失败。这 4 行就是：***重建目录 → 给权限 → 重启 Nginx***

```
[root@master ~]# sudo mkdir -p /run/nginx
[root@master ~]# sudo chown -R nginx:nginx /run/nginx
[root@master ~]# sudo chmod 755 /run/nginx
[root@master ~]# sudo systemctl restart nginx
[root@master ~]# sudo systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since 日 2026-04-26 10:35:58 CST; 18s ago
     Docs: http://nginx.org/en/docs/
  Process: 72101 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 72102 (nginx)
   CGroup: /system.slice/nginx.service
           ├─72102 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           ├─72103 nginx: worker process
           └─72104 nginx: worker process

4月 26 10:35:58 master systemd[1]: Starting nginx - high performance web server...
4月 26 10:35:58 master systemd[1]: Can't open PID file /var/run/nginx.pid (yet?) after start: No ...tory
4月 26 10:35:58 master systemd[1]: Started nginx - high performance web server.
Hint: Some lines were ellipsized, use -l to show in full.
[root@master ~]# 	
[root@master ~]# nginx -v
nginx version: nginx/1.26.1
[root@master ~]# 
```

由于之前升级PHP，Nginx导致系统临时目录/run/nginx丢失，nginx启动时找不到地方写进程所以报**Permission denied**（权限不足），以至于启动失败

到了这里我才发现前面的nginx服务并没有成功，我才知道下面这段话里点击：[跳转到前面那段内容](#anchor-1)

启动成功后去window浏览器输入IP

![image-20260426105336705](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260426105336705.png)

看到欢迎界面，升级成功

由于我是先升级的php再升级的nginx,所以我检查了一下,去浏览器搜索IP/info.php,发现配没有出现界面而是直接下载的info.php文件

问题：典型的 Nginx 升级后配置丢失导致的故障。

原因：我们还原 Nginx 旧配置时，虽然恢复了处理 PHP 的 `location ~ \.php$` 块，但**丢失了一个关键的全局配置指令**，导致 Nginx 根本不把 `/info.php` 识别为需要处理的 PHP 文件。

所以现在需要把PHP的运行用户修改成nginx



```
 [root@master ~]#vi /etc/nginx/nginx.conf
```

将文件中的server注释掉

进入/etc/nginx/conf.d/目录下

```
 [root@master ~]#vi default.conf  #将php的内容前的#删掉
```

![image-20260426115927303](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260426115927303.png)



![image-20260426160733304](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260426160733304.png)

$document_root$fastcgi_script_name。$document_root 会自动指向 root 指令设置的路径，这样 PHP-FPM 就能准确找到文件了。

验证并重载 Nginx：

```
[root@master ~]# sudo nginx -t && sudo systemctl reload nginx
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@master ~]# 
```

去window浏览器查看192.168.227.175/info.php

![image-20260426160801528](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260426160801528.png)

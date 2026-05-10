# 《从零搭建 LNMP 服务器实战记录与排错手册》

# 0. 项目概述

项目目标：独立从零搭建一套可运行现代Web应用的LNMP环境，并记录所有故障处理过程

最终环境版本清单：

| 操作系统 |       CentOS7       |
| :------: | :-----------------: |
|  Nginx   |  1.26.1(官方主线)   |
|  MySQL   | 8.0.46 (官方社区版) |
|   PHP    |   8.2.20(Remi 源)   |

# 1.环境准备

## 1.1 虚拟机配置决策

CPU：配置 2 核 vCPU，满足 LNMP 基础服务并发处理需求，避免单核瓶颈；

内存：分配 2GB 内存，预留 1GB 系统缓存，防止 Nginx/MySQL 内存不足导致 OOM；

磁盘：总容量 20GB，采用 SSD 存储介质，提升数据库读写与静态资源加载速度；

配置原则：兼顾基础性能与资源利用率，符合中小型 Web 应用的硬件基线。

## 1.2 网络规划

网络模式选择：VMware 虚拟机采用桥接模式（Bridged）；

选择原因：

1. 桥接模式下虚拟机可获取与宿主机同网段的独立 IP 地址，局域网内其他设备可直接访问虚拟机的 Web 服务，贴近生产环境的网络部署逻辑；
2. 相比 NAT 模式，无需端口映射即可对外提供服务，简化测试阶段的网络配置；
3. 便于通过 SSH 远程连接虚拟机，模拟生产环境的远程运维场景。

网络验证：配置完成后通过`ip addr`确认 IP 地址与宿主机同网段，`ping 网关IP`验证网络连通性。

## 1.3 系统安装（最小化安装、手动分区方案）

安装方式：CentOS 7.9 最小化安装（Minimal Install），仅保留系统核心组件，减少不必要的进程与安全风险；

手动分区方案（MBR 分区表）：

|   分区   | 挂载点 | 大小  | 文件系统 |                 作用                 |
| :------: | :----: | :---: | :------: | :----------------------------------: |
| 交换分区 |  swap  | 2047M |   swap   | 内存扩展，缓解内存不足场景的系统压力 |
|  根分区  |   /    | 18GB  |   ext4   |  存放系统与 LNMP 所有服务的安装文件  |

安装程序在创建分区时，为了满足对齐要求，会把末尾的 1MiB 空间留出来，导致最终显示 2047 MiB

- 安装验证：系统启动后通过`df -h`检查分区挂载状态，`free -m`确认交换分区生效。

# 2. 系统初始化

## 2.1 网络配置（DHCP 获取 IP）

操作步骤：

1. 编辑网卡配置文件`/etc/sysconfig/network-scripts/ifcfg-ens33`（网卡名以实际为准）；
2. 设置`BOOTPROTO=dhcp`、`ONBOOT=yes`，启用 DHCP 自动获取 IP；
3. 重启网络服务：`systemctl restart network`；



验证：执行`ip addr`查看 IP 分配结果，`ping www.baidu.com`验证公网连通性。

### 2.2 用户与权限（创建普通用户`wangcan`并授权 sudo）

安全原则：避免直接使用 root 用户操作，遵循 “最小权限” 原则；

操作步骤：

- 创建用户：`useradd wangcan`；
- 设置密码：`passwd wangcan`；

问题：root用户创建的wangcan普通用户尚未被加入到系统的特权组，普通用户想要使用sudo 就需要将用户加入允许使用sudo的组中,先回到root用户再执行下面的命令

```
usermod -aG wheel wangcan
```



**故障案例1：CentOS 7 EOL 导致 yum 源失效**

现象：执行下面的命令

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

出现错误提示

<font color="red">Could not resolve host: mirrorlist.centos.org; 未知的错误"    经典的错误，**意思是服务器无法解析域名mirrorlist.centos.org,所以yum找不到，不知道从哪里下载软件**</font>

诊断过程：ping命令发现可以使用外网和公网，所以排除网络连通性问题

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

通过询问AI发现 CentOS 官方公告，确认 CentOS 7 于 2024 年 6 月 30 日结束生命周期（EOL），官方镜像站`mirrorlist.centos.org`停止维护；

检查`/etc/yum.repos.d/CentOS-Base.repo`，发现配置的确实是已失效的官方源地址。

解决方法：

先备份/etc/yum.repos.d文件，再将 CentOS 官方源地址从 mirror.centos.org 改为 vault.centos.org（归档站）

```
[wangcan@master ~]$ sudo cp -r /etc/yum.repos.d /etc/yum.repos.d.backup  #先备份
[sudo] wangcan 的密码：

# 将 CentOS 官方源地址从 mirror.centos.org 改为 vault.centos.org（归档站）
[wangcan@master ~]$ sudo sed -i 's/mirror.centos.org/vault.centos.org/g' /etc/yum.repos.d/CentOS-*.repo
[sudo] wangcan 的密码：
[wangcan@master ~]$ sudo sed -i 's/#baseurl/baseurl/g' /etc/yum.repos.d/CentOS-*.repo
[wangcan@master ~]$ sudo sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*.repo
```

然后清理缓存，重新缓存让新地址生效

```
[wangcan@master ~]$ sudo yum clean all
已加载插件：fastestmirror
正在清理软件源： base extras updates
Cleaning up list of fastest mirrors
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

验证：

安装EPEL仓库成功

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

# 3. 服务部署与排错

## 3.1 Nginx部署与升级

初始安装（EPEL源，版本1.20.1）

安装

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
......
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

#### 故障案例 2：80 端口被系统内置 Apache 占用

- 现象：执行`systemctl start nginx`提示`Job for nginx.service failed because the control process exited with error code`，启动失败；

- 诊断过程：

  1. 查看 Nginx 日志：`cat /var/log/nginx/error.log`，提示`bind() to 0.0.0.0:80 failed (98: Address already in use)`；
  2. 检查端口占用：`netstat -tulpn | grep 80`，发现`httpd`（Apache）进程占用 80 端口；
  3. 排查原因：CentOS 7 最小化安装后默认未安装 Apache，推测是系统初始化阶段误装相关依赖导致。

  

- 解决方法：

  1. 停止并禁用 Apache 服务：`sudo systemctl stop httpd && sudo systemctl disable httpd`；
  2. 卸载 Apache（可选，避免后续冲突）：`sudo yum remove -y httpd`；
  3. 重启 Nginx：`sudo systemctl restart nginx`；
  4. 验证：`netstat -tulpn | grep 80`确认 Nginx 占用 80 端口，访问 IP 可看到 Nginx 默认页面。

#### **故障案例 3：升级至 Nginx 1.26.1 后，因 PID 文件权限导致启动失败**

- **现象**：按照官方流程升级 Nginx 到 1.26.1 版本后，执行 `sudo systemctl start nginx` 失败。查看服务状态，日志中明确提示 `Failed to parse PID from file /var/run/nginx.pid: Invalid argument` 和 `nginx: [emerg] open() "/run/nginx.pid" failed (13: Permission denied)`。
- **诊断过程**：
  1. **定位关键日志**：直接查看 `systemctl status nginx` 的详细输出，发现权限拒绝错误。
  2. **检查 PID 文件路径**：`/run/nginx.pid` 是 Nginx 1.26.1 版本在 CentOS 7 上默认的 PID 文件路径，但新安装的服务单元文件（service unit）有时会存在权限声明不完整的情况。
  3. **定位根因**：Nginx 的 master 进程以 `root` 用户启动，但 PID 文件却因为目录权限问题无法写入，导致启动流程中断。
- **解决方案**：
  1. 手动创建 PID 文件所在目录并赋予正确权限：`sudo mkdir -p /run/nginx`。
  2. 将目录属主改为 Nginx 默认运行用户：`sudo chown -R nginx:nginx /run/nginx`。
  3. 重启 Nginx 服务，恢复正常。
- **运维思维**：**服务启动失败时，第一反应不是去搜“Nginx 启动失败怎么办”，而是查看 `systemctl status` 和服务的专属日志。** 通过日志快速定位到 `Permission denied` 这个核心错误，是排错效率高低的关键。

------

## **3.2 MySQL 8.0 部署**

- **部署步骤**：添加 MySQL 官方 YUM 源，执行 `yum install -y mysql-community-server` 完成安装，启动服务并通过 `systemctl enable mysqld` 设置开机自启。

#### **故障案例 4：GPG 密钥校验失败**

- **现象**：安装 MySQL 时，YUM 在验证软件包签名时报错 `公钥尚未安装`，提示 `失败的软件包是：mysql-community-common-8.0.46-1.el7.x86_64 GPG 密钥配置为：file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql`，安装中断。
- **诊断过程**：
  1. **理解 GPG 机制**：确认这是 Linux 系统的软件包签名校验机制，目的是防止安装被篡改的软件包。
  2. **定位密钥版本**：发现报错指向的本地密钥文件是旧的，而 MySQL 8.0.46 版本的软件包使用了新的签名密钥。
  3. **查找正确密钥**：根据 MySQL 官方文档，找到用于验证新版软件包的正确密钥文件地址。
- **解决方案**：
  执行命令更新本地密钥：`sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023`。之后重新执行安装命令，问题解决。
- **运维思维**：**在对待安全的态度上，不能有“先关掉校验再说”的想法。** 遇到 GPG 校验失败，应主动去寻找并导入正确的密钥，理解这是一种保护机制而非障碍。
- **安全初始化**：安装完成后，使用 `sudo grep "temporary password" /var/log/mysqld.log` 找到临时密码，运行 `sudo mysql_secure_installation` 脚本。按照提示修改密码，并回答 `y` 以移除匿名用户、禁止 root 远程登录、删除测试数据库、重载权限表，完成基线安全配置。

------

## **3.3 PHP 8.2 部署与集成**

- **升级动机**：CentOS 7 默认源的 PHP 版本为 5.4，该版本已于 2015 年停止安全支持，且不支持现代框架（如 Laravel 9+）和新版数据库驱动，必须升级。

#### **故障案例 5：Remi 源网络超时**

- **现象**：添加 Remi 源后，执行 `yum makecache` 或 `yum install` 时，频繁出现 `Timeout on ... remi-*.repo ...` 的错误，下载速度极慢甚至失败。
- **诊断过程**：
  1. **确认网络与 DNS**：`ping 8.8.8.8` 和 `ping www.baidu.com` 均正常，排除基础网络问题。
  2. **定位源地址**：查看错误日志，发现连接的是 Remi 官方源，其 CDN 服务器位于海外，访问延迟高。
- **解决方案**：
  将 Remi 仓库的地址替换为国内清华大学的开源镜像站，通过 `sed` 命令批量修改 `/etc/yum.repos.d/remi*.repo` 文件中的地址。之后重建缓存，速度恢复正常。
- **运维思维**：**在国内部署服务，具备“配置国内镜像源”的意识本身就是一种职业素养。** 这也是一种“成本意识”——时间成本也是成本。

#### **故障案例 6：配置了 `location ~ \.php$`，但 PHP 文件仍被下载**

- **现象**：Nginx 和 PHP-FPM 服务均在运行，访问 `info.php` 时，浏览器不返回执行结果，而是直接下载该 PHP 文件。
- **诊断过程**：
  1. **检查 Nginx 配置**：确认在 `conf.d/default.conf` 中已配置 `location ~ \.php$` 块，且 `fastcgi_pass` 指向了正确的 `127.0.0.1:9000`。
  2. **检查 PHP-FPM 监听**：`netstat -tulnp | grep 9000` 确认 PHP-FPM 正在监听 9000 端口。
  3. **检查请求分发**：用 `curl -I http://ip/info.php` 查看响应头，发现 `Content-Type: application/octet-stream`，证明 Nginx 并未将请求交给 FastCGI 处理。
  4. **定位根因**：经过反复排查，发现是 `SCRIPT_FILENAME` 参数路径配置错误，导致 Nginx 虽然找到了 PHP-FPM，但传递给它的文件路径是错误的，PHP-FPM 找不到文件，最终 Nginx 将文件作为静态内容直接输出。
- **解决方案**：
  将 `fastcgi_param SCRIPT_FILENAME` 的值由错误的静态路径修改为动态变量：`fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;`，重载 Nginx 后问题解决。
- **运维思维**：**“服务在运行”不等于“服务可用”。** 这个故障教会了我，排查问题要像剥洋葱：从服务层（Nginx），到传输层（FastCGI），再到应用层（PHP-FPM），一层层验证。同时，`curl -I` 是排查 Web 服务问题的利器。

------

# **4. 集成验证与安全收尾**

- **集成验证**：在 Nginx 默认网站根目录下创建 `info.php` 文件，写入 `<?php phpinfo(); ?>`。通过浏览器成功访问到紫色 PHP 信息页，验证了 **Nginx -> PHP-FPM -> PHP** 的完整调用链。

- **最终版本确认**：

  - `nginx -v`: nginx/1.26.1

    ![image-20260426105336705](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260426105336705.png)

  - `mysql --version`: mysql Ver 8.0.46

    ![image-20260506142508426](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260506142508426.png)

  - `php --version`: PHP 8.2.20

    ![image-20260426160801528](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\image-20260426160801528.png)

- **安全收尾**：

  - 执行 `rm -f /usr/share/nginx/html/info.php` 删除敏感信息文件。
  - 执行 `systemctl list-unit-files --state=enabled | grep -E 'nginx|mysql|php'`，确认所有服务均已设置为开机自启。

# **5. 项目收获**

这个项目，是我从理论到实践、从“学知识点”到“解决真问题”转变的缩影。回顾整个过程，我最大的收获不是学会了装几个软件，而是理解并初步掌握了一套运维工作的核心逻辑：

1. **方法论重于死记硬背**：我认识到，运维的核心不是能记住多少命令，而是建立一套 **“问题诊断-方案设计-操作执行-验证复盘”** 的闭环思维。面对“yum 源失效”、“端口冲突”、“权限报错”这些问题时，我从最初的慌乱，变得能开始尝试通过查看日志、分析错误信息、查阅文档来一步步定位和解决问题。
2. **记录是最好的老师**：编写这份排错手册的过程，本身就是最好的复盘。每一个被我记录在案的故障案例，都极大地加深了我对操作系统、网络和软件运行机制的理解。这些实践中的踩坑经验，比任何书本知识都来得深刻。
3. **安全意识是底线**：从给普通用户分配 `sudo` 权限、执行 `mysql_secure_installation` 到删除 `info.php` 文件，我开始养成“一切操作，安全先行”的职业习惯。这让我明白，运维工程师是系统和数据的守护者，责任重大。

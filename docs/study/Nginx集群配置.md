# Nginx 集群配置

[TOC]

本文主要是讲一下nginx负载均衡配置和Nginx+keepalived高可用集群方案。

## Nginx负载均衡

​	负载均衡，就是当服务器快要承受不住的时候，加一台服务器来分担压力，请求会分配到两台服务器上，两台服务器上部署相同的内容相当于一个分身，可以处理相同的事情；

### Nginx 安装（Linux）

1. 可以联网的服务器直接用下面的命令即可

```
yum install nginx
```

2. 不能联网的服务器可以用rpm包安装，安装方法可以参考：https://www.jianshu.com/p/29016701ba4a

### Nginx 负载均衡算法详解

​	找到nginx.conf，Linux服务器的nginx.conf配置文件一般在/etc/nginx/conf下，可以用``whereis nginx``命令找到nginx的安装目录。

​	配置upstream，这个配置是写一组被代理的服务器地址，然后配置负载均衡的算法。这里的被代理服务器地址有2中写法。

```xml
upstream myserver { #注意该名称不能带下划线，否则会报错
    server 192.168.10.240:8080; 
    server 192.168.10.241:8080;
}
server {
    ....
    location  / {         
        proxy_pass  http://myserver;  #请求转向myserver 定义的服务器列表         
    }
}
```

#### 算法一：热备

​	热备：如果你有2台服务器，当一台服务器发生事故时，才启用第二台服务器给提供服务。服务器处理请求的顺序：AAAAAAAA….，当A挂了，变成BBBBBBBBB……

```
upstream myserver { 
    server 192.168.10.240:8080;                #服务器A
    server 192.168.10.241:8080; backup;  #热备     #服务器B
}
```

#### 算法二：轮询

轮询：nginx默认就是轮询其权重都默认为1，服务器处理请求的顺序：ABABABABAB....

```
upstream myserver { 
    server 192.168.10.240:8080;  #服务器A
    server 192.168.10.241:8080;  #服务器B   
}
```

#### 算法三：加权轮询

加权轮询：跟据配置的权重的大小而分发给不同服务器不同数量的请求。如果不设置，则默认为1。下面服务器的请求顺序为：ABBABBABBABBABB....

```
upstream mysvr { 
    server 192.168.10.240:8080 weight=1;  #服务器A
    server 192.168.10.241:8080 weight=2;  #服务器B
}
```

#### 算法四：ip_hash

ip_hash：nginx会让相同的客户端ip请求相同的服务器。

```
upstream mysvr { 
    server 192.168.10.240:8080;  #服务器A
    server 192.168.10.241:8080;  #服务器B  
    ip_hash;
}
```



**nginx负载均衡配置的几个状态参数讲解**。

- down，表示当前的server暂时不参与负载均衡。
- backup，预留的备份机器。当其他所有的非backup机器出现故障或者忙的时候，才会请求backup机器，因此这台机器的压力最轻。
- max_fails，允许请求失败的次数，默认为1。当超过最大次数时，返回proxy_next_upstream 模块定义的错误。
- fail_timeout，在经历了max_fails次失败后，暂停服务的时间。max_fails可以和fail_timeout一起使用。



## Nginx+keepalived高可用配置

​	keepalived软件主要是通过VRRP协议实现高可用功能的。在keepalived服务工作时，主Master节点会不断地向备节点发送（多播的方式）心跳消息，用来告诉备Backup节点自己还活着。当主节点发生故障时，就无法发送心跳的消息了，备节点也因此无法继续检测到来自主节点的心跳了。于是就会调用自身的接管程序，接管主节点的IP资源和服务。当主节点恢复时，备节点又会释放主节点故障时自身接管的IP资源和服务，恢复到原来的备用角色。

​	Nginx+keepalived高可用主要在两个Nginx负载均衡服务器器上做高可用配置，Nginx01作为主节点，Nginx02作为备节点。

### 安装keepalived（Linux）

​	keepalived的安装非常简单，直接使用yum来安装即可。	

```text
yum install keepalived -y
```

安装之后，启动keepalived服务，顺便把keepalived写入开机启动的脚本里面去。。

```text
/etc/init.d/keepalived star
echo "/etc/init.d/keepalived start" >>/etc/rc.local
```

启动之后会有三个进程，没问题之后可以关闭keepalived软件，接下来要修改keepalived的配置文件。

### 主节点的配置文件

```xml
global_defs {
    notification_email {
        #mr@mruse.cn       # 指定keepalived在发生切换时需要发送email到的对象，一行一个
        #sysadmin@firewall.loc
    }
    notification_email_from xxx@163.com   # 指定发件人
    smtp_server smtp@163.com              # smtp 服务器地址
    smtp_connect_timeout 30               # smtp 服务器连接超时时间
    router_id LVS_1 # 必填，标识本节点的字符串,通常为hostname,但不一定非得是hostname,故障发生时,邮件通知会用到
    }

vrrp_script chk_nginx {                    #详细看下面
    script "/etc/keepalived/tomcat_check.sh"            #检测服务shell
    interval 2                                    #每个多长时间探测一次
    weight -20                                            #每个多长时间探测一次                                                
}

_instance VI_1 {  # 实例名称
    state MASTER      #  必填，可以是MASTER或BACKUP，不过当其他节点keepalived启动时会将priority比较大的节点选举为MASTER
    interface ens33    #  必填， 节点固有IP（非VIP）的网卡，用来发VRRP包做心跳检测
    mcast_src_ip 172.16.1.139 #本机的ip，需要修改
    virtual_router_id 101 #  必填，虚拟路由ID,取值在0-255之间,用来区分多个instance的VRRP组播,同一网段内ID不能重复;主备必须为一样;
    priority 100      #  必填，用来选举master的,要成为master那么这个选项的值最好高于其他机器50个点,该项取值范围是1-255(在此范围之外会被识别成默认值100)
    advert_int 1      #  必填，检查间隔默认为1秒,即1秒进行一次master选举(可以认为是健康查检时间间隔)
    authentication {  #  必填，认证区域,认证类型有PASS和HA（IPSEC）,推荐使用PASS(密码只识别前8位)
        auth_type PASS  # 默认是PASS认证
        auth_pass 1111 # PASS认证密码
    }
    virtual_ipaddress {
        172.16.1.100    #  必填，虚拟VIP地址,允许多个
    }
    track_script {         # 检测shell                            
        chk_nginx
    }
}
```

### 备节点的配置文件

```text
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_2
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 146
    mcast_src_ip 172.16.1.79
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    track_script{
        chk_nginx
        }

    virtual_ipaddress {
        172.16.1.100
    }
}
```

注解：修改配置文件主要就是上面加粗的几个地方，下面说明一下那几个参数的意思：

- router_id：路由标识，在一个局域网里面应该是唯一的，**两个配置文件的标识必须一致**；
- vrrp_instance VI_1{...}：这是一个VRRP实例，里面定义了keepalived的主备状态、接口、优先级、认证和IP信息；
- state：定义了VRRP的角色，interface定义使用的接口，这里我的服务器用的网卡都是eth1,根据实际来填写，virtual_router_id是虚拟路由ID标识，一组的keepalived配置中主备都是设置一致，priority是优先级，数字越大，优先级越大，auth_type是认证方式，auth_pass是认证的密码
- virtual_ipaddress ｛...：定义虚拟IP地址，可以配置多个IP地址，这里我定义为192.168.31.5，绑定了eth1的网络接口，虚拟接口eth1:1
- priority：优先级，**主机必须比备用机优先级高**



### 编辑chk_nginx脚本

```
#!/bin/bash
A=`ps -C nginx –no-header |wc -l`
if [ $A -eq 0 ];then
    /home/admin/nginx/sbin/nginx
    sleep 2
    if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
        killall keepalived
    fi
fi

```



**修改好主节点之后，保存退出，然后启动keepalived，几分钟内会生成一个虚拟IP：192.168.31.5**



然后修改备节点的配置文件，保存退出后启动keepalived，不会生成虚拟IP，如果生成那就是配置文件出现了错误。备节点和主节点争用IP资源，这个现象叫做“裂脑”。




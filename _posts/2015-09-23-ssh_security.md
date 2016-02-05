---
layout: post
title: Enhance ssh service
description: 
category: 网络安全
subtitle:
author: "fishcried"
header-img: "img/bg/home-bg.jpg"
tags: [ssh]
---

`grep "Failed password " /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -nr`

  这条命令的意图是简单统计出ssh登录本机失败的情况,并根据次数做了简单的排序。不管主机运行在内网还是直接暴露在外网，结果都可能会让人意外。ssh暴力破解如此常见且有效：

1. 如果一个服务能够进行暴力破解，那么对方的安全意识应该会很差，自然而然的密码也可能会很简单，而且开发者，管理员，安全人员为了便利往往会设置很简单的密码。
2. ssh暴击破解攻击成本很低很低，现在的自动化工具非常的好用，比如hydra,medusa,metasploit都非常的智能化。使用者不需要了解原理，只要瞄准目标，按下回车。

  本文整理ssh服务的常用的加固手段。

# SSH服务安全加固常见手段

作为关键的老牌服务,ssh的相关强化手段已经非常固定了,下面作下梳理。

## 修改服务默认端口

**优点**

简单的一个端口修改，增加的攻击难度是非常大的。因为攻击前肯定要进行探测，通常会有针对性的手动探测查最常用的端口，降低暴露风险，但是端口改掉了只能让攻击者无奈。 如果攻击者进行大规模扫描的话，那么会就触发检测设备。没有谁愿意这样。而高度自动化傻瓜话的攻击通常只识别默认端口. 所以一个简单的改动大大提高了攻击门槛，可避免很多的噪声攻击。

**缺点**

这玩意缺点也很明显，用起来不方便。自己的系统倒是没有什么，如果做成镜像给云用户，那么用户搞了半天都登录不上来，又不看说明，最后会留下不好的感受。


## 禁止root远程登录

**优点**

这个是肯定要做的。因为是linux系统就有root用户,所以用户名不用猜，只需要猜密码就可以了。平均要猜解上千次的过程，现在可能需要几十次就可以了。而且root这个用户被爆破了整个系统就沦陷了。如果是其他的用户普通用户，攻击者还存在提权的关卡。

**缺点**

无

## 只开放密钥认证方式

**优点**

密码还能猜，密钥基本没有猜测的可能。而且密钥结合着密码欢迎任性者来爆破。

**缺点**

密钥不好保存,用户多地登录不是很方便。所以个人用户往往会把ssh配置改掉，自己开启了密码认证。制作密钥的时候加密码，两个结合使用应该会很好。

## 使用Denyhosts自动维护攻击者名单

Denyhosts原理是通过分析系统的认证日志，根据其中ip的登录失败次数来判断是否需要将其加入黑名单(TCP Wrapper功能),进而防御暴力攻击。而且它可以联动iptables,自动的发送邮件，自动同步云端黑名单。

**优点**

Denyhosts提供了一种主动一点的防御方式，到了异常临界点会将攻击者加入黑名单，不需要管理员参与，很便利。当有Denyhosts的情况下,即使密码很简单，即使root可以直接登录,但只要不是3,4次就能猜到的情况下,Denyhosts就可以抵御住。

**缺点**

由于Denyhosts接管了黑名单功能，一旦用户误操作导致自己被加入了黑名单就需要管理员去解除，如果管理员不够专业那就少不了麻烦。

## 添加防火墙规则

iptables的recent,limit模块可以根据一个时间段内同一ip发起的连接数来判断是否存在异常行为，从而进行日志记录，DROP，限流等。

**优点**

在主机业务清晰的情况下，防火强规则可以很有效的阻止攻击者。

**缺点**

对用户要求较高。

## 去除ssh flag信息

nc或者telnet可以很容易的获取ssh的版本信息，最好把flag去除，不过需要重新编译openssh，暂不考虑.

# SSH安全加固实录

## 规范SSH服务端配置 `/etc/ssh/sshd_config`

    cat /etc/ssh/sshd_config
    # 修改默认的端口
    Port 10022
    # 使用version2
    Protocol 2

    # 发起连接后到成功登录之间的timeout为60s
    LoginGraceTime 60
    # 禁止root登录
    PermitRootLogin no
    StrictModes yes
    PermitEmptyPasswords no
    # 关闭password认证
    PasswordAuthentication no
    # 开启密钥认证
    PubkeyAuthentication yes

    IgnoreRhosts yes
    RhostsRSAAuthentication no
    HostbasedAuthentication no

    IgnoreUserKnownHosts no

    ChallengeResponseAuthentication no
    # 使用PAM,PAM可以做很多事情，后续会进行详细的配置
    UsePAM yes
    # 关闭转发,有需求的话才开启
    X11Forwarding no

    # 啥都不输出,信息越少越好
    PrintMotd no
    PrintLastLog no

    TCPKeepAlive yes
    UsePrivilegeSeparation yes

    # 发起连接但是没有成功登入状态的最大连接数
    MaxStartups 10
    # 一个连接中认证失败的最大次数
    MaxAuthTries 2

    # 记录日志
    SyslogFacility AUTH
    LogLevel INFO

    HostKey /etc/ssh/ssh_host_rsa_key
    HostKey /etc/ssh/ssh_host_dsa_key
    HostKey /etc/ssh/ssh_host_ecdsa_key
    HostKey /etc/ssh/ssh_host_ed25519_key

    # 功能性配置,防止客户端长时间无操作断线
    ClientAliveInterval 60
    ClientAliveCountMax 10

    AcceptEnv LANG LC_*
    Subsystem sftp /usr/lib/openssh/sftp-server

以上配置中进行注视的为关键配置。主要做了:

1. 修改默认端口
2. 禁止root登录
3. 只允许密钥认证
4. 限制每个连接中的认证失败的次数，该配置并不能解决暴力猜解的问题。

## 配备denyhosts

denyhosts没有被ubuntu加入官方源,无法通过apt-get来安装，只能通过源码安装，而且默认支持的centos,所以没有upstart scripts，需要进行手动的修改。denyhosts的最新代码已经在github上托管，不再用sourceforge. 最新版本为v3.0.

**安装denyhosts**

    # 下载代码
    mkdir /tmp/denyhosts
    cd /tmp/denyhosts
    wget https://github.com/denyhosts/denyhosts/archive/v3.0.tar.gz
    tar -zxf v3.0.tar.gz
    cd denyhosts-3.0/
    python setup.py install
    # 安装后会自动将配置文件cp到/etc下，我们不用这个，所以删掉
    rm /etc/denyhosts.conf

**配置denyhosts**

    # 准备配置文件
    mkdir /etc/denyhosts 
    cat > /etc/denyhosts/denyhosts.conf <<EOF
    SECURE_LOG = /var/log/auth.log 
    HOSTS_DENY = /etc/hosts.deny
    ETC_DIR = /etc/denyhosts
    LOCK_FILE = /var/run/denyhosts.pid
    DAEMON_LOG = /var/log/denyhosts
    WORK_DIR = /var/lib/denyhosts
    PURGE_DENY = 30d  
    PURGE_THRESHOLD = 2 
    BLOCK_SERVICE = sshd
    DENY_THRESHOLD_INVALID = 5
    DENY_THRESHOLD_VALID = 4
    DENY_THRESHOLD_ROOT = 3
    DENY_THRESHOLD_INVALID = 5
    SUSPICIOUS_LOGIN_REPORT_ALLOWED_HOSTS=NO
    HOSTNAME_LOOKUP=NO
    ALLOWED_HOSTS_HOSTNAME_LOOKUP=NO
    AGE_RESET_VALID=1d
    AGE_RESET_INVALID=3d
    AGE_RESET_ROOT=1d
    DAEMON_SLEEP= 10m
    DAEMON_PURGE = 1h
    RESET_ON_SUCCESS = yes
    DENY_THRESHOLD_RESTRICTED=1
    EOF

    # 准备服务脚本
    cp daemon-control-dist daemon-control
    sed -i '/^DENYHOSTS_BIN/s:/usr/sbin/denyhosts:/usr/local/bin/denyhosts:' daemon-control
    sed -i '/^DENYHOSTS_CFG/s:/etc/denyhosts.conf:/etc/denyhosts/denyhosts.conf:' daemon-control
    sed -i '2,8d' daemon-control
    sed -i '1a\
    ### BEGIN INIT INFO\
    # Provides:          denyhosts\
    # Default-Start:     2 3 4 5\
    # Default-Stop:      0 1 6\
    # Required-Start:    $local_fs $remote_fs\
    # Required-Stop:     $local_fs $remote_fs\
    # Short-Description: Bring up/down the DenyHosts daemon\
    # Description:       Activates/Deactivates the DenyHosts daemon to block\
    #                    ssh attempts\
    ### END INIT INFO' daemon-control
    cp daemon-control /etc/init.d/denyhosts
    # 添加开机启动
    update-rc.d denyhosts defaults 80 20


**配置项目说明**

- SECURE_LOG = /var/log/auth.log 
- HOSTS_DENY = /etc/hosts.deny
- PURGE_DENY = 30d  
    30天后清除以阻止的ip
- PURGE_THRESHOLD = 2 
   黑名单中的host最多被解除的次数,超过了就永不移除.再一再二不再三
- BLOCK_SERVICE = sshd
- DENY_THRESHOLD_INVALID = 5
   允许无效用户登录失败的次数
- DENY_THRESHOLD_VALID = 4
   允许有效用户登录失败的次数
- DENY_THRESHOLD_ROOT = 3
   允许root用户登录失败的次数
- WORK_DIR = /var/lib/denyhosts
   工作目录，该目录下记载着分析数据。如果需要解除某ip,需要在该目录下做相应的修改。
- ETC_DIR = /etc/denyhosts
   配置文件目录
- SUSPICIOUS_LOGIN_REPORT_ALLOWED_HOSTS=NO
- HOSTNAME_LOOKUP=NO
- LOCK_FILE = /var/run/denyhosts.pid
- IPTABLES = /sbin/iptables
   与防火墙进行联动。建议不配置，防止系统失控。
- ALLOWED_HOSTS_HOSTNAME_LOOKUP=NO
- AGE_RESET_VALID=1d
  有效用户登录失败记录多久清零
- AGE_RESET_INVALID=3d
- AGE_RESET_ROOT=1d
- DAEMON_LOG = /var/log/denyhosts
- RESET_ON_SUCCESS = yes
  用户成功登录后,清除其失败记录.
- DENY_THRESHOLD_RESTRICTED=1
- DAEMON_SLEEP= 10m
  每次读取认证日志的间隔
- DAEMON_PURGE = 1h
  清除机制的时间粒度

## 添加防火墙规则

在主机的业务以及声明周期明确的情况下,防火墙规则很好添加，而且使用白名单的策略。但是如果在一个通用镜像里添加防火墙规则是很难的事情，因为未来主机用途是什么，生命周期如何都是未知的，所以只能简单的进行黑名单限制。

**白名单策略**

    iptables -F
    iptables -P INPUT DROP
    iptables -P OUTPUT ACCEPT
    iptables -P FORWARD ACCEPT

    iptables -A INPUT -p all -m state --state ESTABLISHED,RELATED -j ACCEPT
    iptables -A INPUT -p all -m state --state INVALID -j DROP
    iptables -A INPUT -p tcp --syn -m state --state NEW \
                      -m multiport --dports 22,80,3306 -j ACCEPT
    iptables -A INPUT -m icmp --icmp-type 8 -j ACCEPT
    iptables -A INPUT -m icmp --icmp-type 11 -j ACCEPT
    iptables -A INPUT -p tcp --dport 22  -m state --state NEW -m recent --name ssh --set
    iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --second 30 --hitcount 20 --name ssh --update -j DROP

- 只允许了22,80,3306端口。
- 允许ping请求与应答。
- 30秒内ssh请求了20次则DROP。

**黑名单策略**

    iptables -F
    iptables -P INPUT ACCEPT
    iptables -P OUTPUT ACCEPT
    iptables -P FORWARD ACCEPT

    iptables -A INPUT -p all -m state --state INVALID -j DROP
    iptables -A INPUT -p all -m state --state ESTABLISHED,RELATED -j ACCEPT
    iptables -A INPUT -p tcp --dport $DIB_SSH_PORT -m state --state NEW -m recent --name ssh --set
    iptables -A INPUT -p tcp --dport $DIB_SSH_PORT -m state --state NEW -m recent --second 30 --hitcount 20 --name ssh --update -j DROP

- 拒绝INVALID状态的请求。可以DROP掉一些畸形数据包
- 30秒内ssh请求了20次则DROP。

# 变更记录

|Why | Who | When |
|----|-----|------|
|创建文件,提醒日后完善内容|fishcired|2015-09-21|
|整理,完善|fishcired|2015-09-23 |

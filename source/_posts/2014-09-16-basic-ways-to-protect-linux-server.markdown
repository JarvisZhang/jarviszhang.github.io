---
layout: post
title: "关于Linux服务器安全防护的基本方法"
date: 2014-09-16 13:38:08 +0800
comments: true
categories: Linux ssh 安全
---

#楔子
服务器的安全防护一直以来都是老生长谈的话题，虽然我们平时接触的服务器影响力一般，不会被攻击者处心积虑利用各种漏洞攻击，但做一些基本的防护措施还是很有必要的，可以避免那些常年活跃在互联网上的自动化攻击。

曾有国外安全研究者做过实验，他们搭建了一台蜜罐服务器，该服务上安装了修改后的SSHD版本，记录所有的登陆尝试和存储的所有会话，一旦被黑客攻击，可以查看到所有暴力破解尝试记录，[实验结果](http://blog.sucuri.net/2013/07/ssh-brute-force-the-10-year-old-attack-that-still-persists.html)也非常有趣。

SSH暴力破解大约自linux系列产品诞生之后，就衍生出来的一种攻击行为，不仅仅SSH暴力破解，ftp、telnet、smtp、mysql等等都是暴力美学黑客的最爱。

本文总结了一些Linux服务器的基本防护方法，可以有效应对大部分普通网络攻击。实验操作系统版本为CentOS 6

#修改ssh默认端口22

众所周知，ssh的默认端口为22，这成为了攻击者的首要目标，同时也会引入大量的错误日志。TCP/IP协议中的端口，端口号的范围从0到65535，我们可以指定一个作为ssh登陆端口，注意不要与其它服务冲突。

ssh服务配置文件为/etc/ssh/sshd_config，可以看到端口号指定的配置被注释了，默认为22

> \#Port 22
	
在配置文件中增加一行`Port 12345`后，重启ssh服务`service sshd restart`，此时，ssh端口改为了12345，登陆时需要额外指定`ssh -p 12345 mytestuser@mytestserver`。

#使用Fail2Ban

fail2ban是一个通过监控日志，防止系统密码被暴力破解的工具，它支持大部分常用服务，如sshd, apache, qmail, proftpd, sasl, asterisk等等，当发现服务器有被暴力破解的迹象时，fail2ban会采取多种手段进行应对，如iptables, tcp-wrapper, shorewall, mail notifications等等。

###安装依赖

#####必选

- Python >= 2.4

#####可选
- iptables
- shorewall
- tcp-wrappers
- a working mail command
- Gamin the File Alteration Monitor

###安装步骤

1. 官网下载地址

	http://www.fail2ban.org/wiki/index.php/Downloads

2. 解压及安装
		
		[root@jsi-dell13 downloads]# tar xzvf 0.8.14.tar.gz
		[root@jsi-dell13 downloads]# cd fail2ban-0.8.14/
		[root@jsi-dell13 fail2ban-0.8.14]# ./setup.py install
		[root@jsi-dell13 fail2ban-0.8.14]# cp files/redhat-initd /etc/init.d/fail2ban
		[root@jsi-dell13 fail2ban-0.8.14]# chmod 755 /etc/init.d/fail2ban 
	
3. 配置开机启动

		[root@jsi-dell13 fail2ban-0.8.14]# ln -s /etc/init.d/fail2ban /etc/rc2.d/S20fail2ban
		
4. 配置日志轮询

	创建文件/etc/logrotate.d/fail2ban，写入
	
	> /var/log/fail2ban.log {
    
    > weekly
    
    > rotate 7
    
    > missingok
    
    > compress
    
    > postrotate
    
    > /usr/bin/fail2ban-client set logtarget /var/log/fail2ban.log >/dev/null
    
    > endscript
	
	> }
		
5. 防止ssh字典攻击

	修改配置文件/etc/fail2ban/jail.conf如下
	
	> [ssh-iptables]
	
	>
	
	> \#enabled  = false

	> enabled  = true
	
	> filter   = sshd
	
	> action   = iptables[name=SSH, port=ssh, protocol=tcp]
    
    > &emsp;&emsp;&emsp;&emsp;sendmail-whois[name=SSH, dest=you@example.com,

    > &emsp;&emsp;&emsp;&emsp;sender=fail2ban@example.com, sendername="Fail2Ban"]

	> \#logpath  = /var/log/sshd.log
	
	> logpath  = /var/log/secure

	> maxretry = 5

	以上配置的含义为对于在(findtime)600s内输错5次密码的ip，使用iptables将该ip屏蔽(bandtime)600s。其中findtime和bandtime均可以通过配置修改。详情参见fail2ban[官方手册](http://www.fail2ban.org/wiki/index.php/MANUAL_0_8)
	
#禁止root用户通过ssh方式登陆

root用户几乎是所有类UNIX系统中都存在的用户，攻击者也常常使用root用户暴力攻击，如果不禁止root远程ssh登陆，攻击者获取密码只是时间问题。

编辑文件/etc/ssh/sshd_config，修改配置选项为`PermitRootLogin no`，重启ssh服务`service sshd restart`

提示：请确保服务器上还有其它可ssh登录的用户，创建用户步骤如下：

	[root@JSI-iDPL02 ~]# useradd jsitest
	[root@JSI-iDPL02 ~]# passwd jsitest

#使用公钥密钥的认证方式

RSA加密算法是一种非对称加密算法。在公开密钥加密和电子商业中RSA被广泛使用。RSA是1977年由罗纳德·李维斯特（Ron Rivest）、阿迪·萨莫尔（Adi Shamir）和伦纳德·阿德曼（Leonard Adleman）一起提出的。当时他们三人都在麻省理工学院工作。RSA就是他们三人姓氏开头字母拼在一起组成的。

1973年，在英国政府通讯总部工作的数学家克利福德·柯克斯（Clifford Cocks）在一个内部文件中提出了一个相同的算法，但他的发现被列入机密，一直到1997年才被发表。

对极大整数做因数分解的难度决定了RSA算法的可靠性。换言之，对一极大整数做因数分解愈困难，RSA算法愈可靠。假如有人找到一种快速因数分解的算法的话，那么用RSA加密的信息的可靠性就肯定会极度下降。但找到这样的算法的可能性是非常小的。今天只有短的RSA钥匙才可能被强力方式解破。到2013年为止，世界上还没有任何可靠的攻击RSA算法的方式。只要其钥匙的长度足够长，用RSA加密的信息实际上是不能被解破的。

具体步骤为：

1. 生成公钥密钥

		[root@JSI-iDPL02 ~]# ssh-keygen -C "jsitest@JSI-iDPL02" -t rsa -b 2048 -f jsitest-JSI-iDPL02
		Generating public/private rsa key pair.
		Enter passphrase (empty for no passphrase): 
		
		Enter same passphrase again: 
		
		Your identification has been saved in jsitest-JSI-iDPL02.
		
		Your public key has been saved in jsitest-JSI-iDPL02.pub.
		The key fingerprint is:
		96:bb:d0:67:11:c8:dc:be:bd:6d:00:7c:59:63:e3:3a jsitest@JSI-iDPL02
		The key's randomart image is:
		+--[ RSA 2048]----+
		|                 |
		|       o o    =  |
		|        +.o  = o |
		|         oo.o .  |
		|        S oo .   |
		|       o . +E    |
		|      . o + .o   |
		|       . +   o.  |
		|        .   ...  |
		+-----------------+
		
	Enter passphrase的输入为加密所用的字符串，可以视安全强度决定是否置空。
	
	各参数含义为：
	
	- -C 是对密钥的一个说明，有助于区分不同的密钥用途。
	
	- -t 和 -b 分别指定要生成的密钥类型和密钥长度。

	- -f 指定生成的密钥对文件名。公钥文件名为jsitest-JSI-iDPL02.pub，私钥为jsitest-JSI-iDPL02。
	
	更多参数请参考ssh-keygen手册

2. 当前目录下分别生成了公钥jsitest-JSI-iDPL02.pub以及密钥jsitest-JSI-iDPL02，将**密钥拷贝至客户端**备用，将**公钥拷贝至服务器端**备用

3. 服务器端导入公钥，将对应的公钥加入到服务器上**需要登录的用户**的home目录下的.ssh/authorized_keys文件中

		[root@JSI-iDPL02 ~]# cd /home/jsitest/
		[root@JSI-iDPL02 jsitest]# mkdir .ssh
		[root@JSI-iDPL02 ~]# cat jsitest-JSI-iDPL02.pub >> /home/jsitest/.ssh/authorized_keys

	事实上authorized_keys文件的准确名字是由 sshd_config 中的 AuthorizedKeysFile 配置指定给定的。man sshd_config 获取详细的说明信息。
	
	注意：authorized_keys文件及其所在的目录以及父目录（一直上溯到该用户的HOME目录为止）的权限必须设置为不能被组和其他人写（可以通过`chmod og-w`确认），否则其他人只需要修改这个文件即可以以该身份登录到系统上。
	
4. OpenSSH服务端配置
	
	OpenSSH服务器端配置文件一般为/etc/ssh/sshd_config，和公钥认证有关的两个配置项是：
	
		#RSAAuthentication yes
		#PubkeyAuthentication yes
	
	其缺省值一般为 yes。如果希望仅打开公钥认证，禁用其他的认证方式，则可以修改下列配置项：
	
		PasswordAuthentication no
		ChallengeResponseAuthentication no	
		UsePAM no
		
	重启ssh服务使修改生效
	
		service sshd restart
		
5. 用户登录
	
	对 OpenSSH 客户端，可以通过`ssh -i jsitest-JSI-iDPL02 jsitest@JSI-iDPL`快速测试是否工作。
	
	更一般的方法是将私钥文件jsitest-JSI-iDPL02拷贝至客户端.ssh/目录下，设置.ssh/config配置对不同主机和用户进行配置。如下例：
	
		Host idpl3  							    #别名
		    HostName 10.4.9.191  				    #完整的域名
		    User jsitest  						    #登录该域名使用的账号名
			IdentityFile ~/.ssh/jsitest-JSI-iDPL03 #私钥文件的路径

	这样就可以简化登陆服务器的过程了：
		
		[root@JSI-iDPL03 ~]# ssh idpl2
		Enter passphrase for key '/root/.ssh/jsitest-JSI-iDPL02': 
		Last login: Wed Sep 17 14:38:05 2014 from jsi-idpl03
		[jsitest@JSI-iDPL02 ~]$

#禁用ping
很久以前，一部分操作系统(例如win95)，不能很好处理过大的Ping包，在当时，大部分电脑无法处理大于IPv4最大封包大小（65,535字节）的ping封包。因此发送这样大小的ping可以令目标电脑崩溃。导致出现了Ping to Death的攻击方式(用大Ping包搞垮对方或者塞满网络)，随着操作系统的升级，网络带宽的升级、计算机硬件的升级，大Ping包基本上没有很大的攻击效果(分布式攻击除外)，如果一定要使用Ping包去攻击别的主机，除非是利用TCP/IP协议的其他特性或者网络拓扑结构的缺陷放大攻击的力度(所谓正反馈)，就是俗称的洪水ping攻击。

不过，大部分机构，特别是一些超算中心还是会选择将服务器的ping功能禁掉，以避免不必要的麻烦。

禁用的方法有两种：

- 修改文件/proc/sys/net/ipv4/icmp_echo_ignore_all

	不过这种方法只对ipv4有效，ipv6不提供此功能。
	
		echo 1 > /proc/sys/net/ipv4/icmp_echo_ignore_all
	
- 添加iptables规则
	
	对于ipv4，在/etc/sysconfig/iptables中增加：
	
	> -A INPUT -j REJECT --reject-with icmp-host-prohibited
	
	重启iptables
	
		service iptables restart

	ipv6的配置文件是/etc/sysconfig/ip6tables，添加规则：
	
	> -A INPUT -j REJECT --reject-with icmp6-adm-prohibited
	
	重启ip6tables
		
		service ip6tables restart

至此，服务器不会相应任何ping消息。

#限制用户的su操作

除了在外部做好防御边界，服务器内部同样需要做好权限管制，当用户需要对服务器进行一些权限较高的维护时，需要通过su切换到root用户，所以为了减少安全隐患，需要限制非管理员的su提权操作。

Linux中有wheel组和staff组的概念，wheel组就类似于管理员的组，只有在这个组中的用户才能进行su操作，否则即使知道root密码，也不能通过su命令切换到root用户。具体步骤为：

1. 修改/etc/pam.d/su文件，将`auth            required        pam_wheel.so use_uid`的注释去掉。
2. 修改/etc/login.defs文件，增加一行`SU_WHEEL_ONLY yes`配置
3. 将新用户jsitest添加到wheel组中
		
		usermod -G wheel jsitest
		
至此，只有在wheel组中的用户才能通过su切换至root用户

#Reference
[1] 四大Linux服务器攻击方式及防范策略[EB/OL]. http://www.enet.com.cn/article/2012/0815/A20120815150578.shtml

[2] SSH Brute Force – The 10 Year Old Attack That Still Persists[EB/OL].  Daniel Cid. http://blog.sucuri.net/2013/07/ssh-brute-force-the-10-year-old-attack-that-still-persists.html

[3] 伪装攻击IP地址的洪水Ping攻击详解[EB/OL]. http://www.edu.cn/sqt_9968/20110318/t20110318_589483.shtml

[4] SSH 公钥认证[EB/OL]. http://blog.knownsec.com/2012/05/ssh-%E5%85%AC%E9%92%A5%E8%AE%A4%E8%AF%81/

[5] fail2ban Main Page[EB/OL]. http://www.fail2ban.org/wiki/index.php/Main_Page

[6] 使用fail2ban防止暴力破解ssh及vsftpd密码[EB/OL]. https://www.centos.bz/2012/04/prevent-ssh-break-in-with-fail2ban/

[7] RSA加密算法[EB/OL]. http://zh.wikipedia.org/wiki/RSA%E5%8A%A0%E5%AF%86%E6%BC%94%E7%AE%97%E6%B3%95

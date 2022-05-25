# kubernetes ansible

[TOC]

#  1.高可用Kubernetes集群规划

集群规划根据自己的情况设置，下面标出了使用需要修改的地方，加以标注，具体内容参考博客内容

| 角色    | 机器名                       | 机器配置 | ip地址                         | 安装软件                                                     |
| ------- | ---------------------------- | -------- | ------------------------------ | ------------------------------------------------------------ |
| ansible | ansible-server.example.local | 2C2G     | 172.31.3.100                   | ansible                                                      |
| master1 | k8s-master01.example.local   | 2C4G     | 172.31.3.101                   | chrony-client、docker、kube-controller-manager、kube-scheduler、kube-apiserver、kubelet、kube-proxy、kubectl |
| master2 | k8s-master02.example.local   | 2C4G     | 172.31.3.102                   | chrony-client、docker、kube-controller-manager、kube-scheduler、kube-apiserver、kubelet、kube-proxy、kubectl |
| master3 | k8s-master03.example.local   | 2C4G     | 172.31.3.103                   | chrony-client、docker、kube-controller-manager、kube-scheduler、kube-apiserver、kubelet、kube-proxy、kubectl |
| ha1     | k8s-ha01.example.local       | 2C2G     | 172.31.3.104 172.31.3.188(vip) | chrony-server、haproxy、keepalived                           |
| ha2     | k8s-ha02.example.local       | 2C2G     | 172.31.3.105                   | chrony-server、haproxy、keepalived                           |
| harbor1 | k8s-harbor01.example.local   | 2C2G     | 172.31.3.106                   | chrony-client、docker、docker-compose、harbor                |
| harbor2 | k8s-harbor02.example.local   | 2C2G     | 172.31.3.107                   | chrony-client、docker、docker-compose、harbor                |
| etcd1   | k8s-etcd01.example.local     | 2C2G     | 172.31.3.108                   | chrony-client、docker、etcd                                  |
| etcd2   | k8s-etcd02.example.local     | 2C2G     | 172.31.3.109                   | chrony-client、docker、etcd                                  |
| etcd3   | k8s-etcd03.example.local     | 2C2G     | 172.31.3.110                   | chrony-client、docker、etcd                                  |
| node1   | k8s-node01.example.local     | 2C4G     | 172.31.3.111                   | chrony-client、docker、kubelet、kube-proxy                   |
| node2   | k8s-node02.example.local     | 2C4G     | 172.31.3.112                   | chrony-client、docker、kubelet、kube-proxy                   |
| node3   | k8s-node03.example.local     | 2C4G     | 172.31.3.113                   | chrony-client、docker、kubelet、kube-proxy                   |

软件版本信息和Pod、Service网段规划：

| **配置信息**       | 备注                                             |
| ------------------ | ------------------------------------------------ |
| 支持的操作系统版本 | CentOS 7.9/stream 8、Rocky 8、Ubuntu 18.04/20.04 |
| Docker版本         | 20.10.14                                         |
| Containerd版本         | 1.5.11                                        |
| kubernetes版本     | 1.23.6                                          |
| Pod网段            | 192.168.0.0/12                                   |
| Service网段        | 10.96.0.0/12                                     |

# 2.安装ansible和配置

## 2.1 安装ansible

```bash
#CentOS
[root@ansible-server ~]# yum -y install ansible

[root@ansible-server ~]# ansible --version
ansible 2.9.25
  config file = /data/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Oct 14 2020, 14:45:30) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]

#ubuntu18.04安装最新版的ansible
root@ubuntu1804:~# apt update

root@ubuntu1804:~# apt -y install software-properties-common

root@ubuntu1804:~# apt-add-repository --yes --update ppa:ansible/ansible

root@ubuntu1804:~# apt -y install ansible
root@ubuntu1804:~# ansible --version
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.17 (default, Feb 27 2021, 15:10:58) [GCC 7.5.0]

#ubuntu 20.04安装
[root@ubuntu ~]# apt -y install ansible
```

## 2.2 配置ansible

```bash
[root@ansible-server ~]# mkdir /data/ansible
[root@ansible-server ~]# cd /data/ansible

[root@ansible-server ansible]# vim ansible.cfg
[defaults]
inventory      = ./inventory
forks          = 10
roles_path    = ./roles
remote_user = root

#下面的IP根据自己的k8s集群主机规划设置
[root@ansible-server ansible]# vim inventory 
[master]
172.31.3.101 hname=k8s-master01
172.31.3.102 hname=k8s-master02
172.31.3.103 hname=k8s-master03

[ha]
172.31.3.104 hname=k8s-ha01
172.31.3.105 hname=k8s-ha02

[harbor]
172.31.3.106 hname=k8s-harbor01
172.31.3.107 hname=k8s-harbor02

[etcd]
172.31.3.108 hname=k8s-etcd01
172.31.3.109 hname=k8s-etcd02
172.31.3.110 hname=k8s-etcd03

[node]
172.31.3.111 hname=k8s-node01
172.31.3.112 hname=k8s-node02
172.31.3.113 hname=k8s-node03

[all:vars]
domain=example.local

[k8s_cluster:children]
master
node

[chrony_server:children]
ha

[chrony_client:children]
master
node
harbor
etcd

[keepalives_master]
172.31.3.104

[keepalives_backup]
172.31.3.105

[haproxy:children]
ha

[master01]
172.31.3.101
```

# 3.设置客户端网卡名和ip

IP地址需要自行设置，可以使用我的初始化脚本设置，CentOS、Ubuntu、Rocky系统初始化脚本：https://raymond.blog.csdn.net/article/details/122162932

```bash
#rocky8和centos系统设置
[root@172 ~]# bash reset.sh 

************************************************************
*                      初始化脚本菜单                      *
* 1.禁用SELinux               12.修改IP地址和网关地址      *
* 2.关闭防火墙                13.设置主机名                *
* 3.优化SSH                   14.设置PS1和系统环境变量     *
* 4.设置系统别名              15.禁用SWAP                  *
* 5.1-4全设置                 16.优化内核参数              *
* 6.设置vimrc配置文件         17.优化资源限制参数          *
* 7.设置软件包仓库            18.Ubuntu设置root用户登录    *
* 8.Minimal安装建议安装软件   19.Ubuntu卸载无用软件包      *
* 9.安装邮件服务并配置邮件    20.重启系统                  *
* 10.更改SSH端口号            21.关机                      *
* 11.修改网卡名               22.退出                      *
************************************************************

请选择相应的编号(1-21): 11
Rocky 8.5 网卡名已修改成功，请重新启动系统后才能生效!

************************************************************
*                      初始化脚本菜单                      *
* 1.禁用SELinux               12.修改IP地址和网关地址      *
* 2.关闭防火墙                13.设置主机名                *
* 3.优化SSH                   14.设置PS1和系统环境变量     *
* 4.设置系统别名              15.禁用SWAP                  *
* 5.1-4全设置                 16.优化内核参数              *
* 6.设置vimrc配置文件         17.优化资源限制参数          *
* 7.设置软件包仓库            18.Ubuntu设置root用户登录    *
* 8.Minimal安装建议安装软件   19.Ubuntu卸载无用软件包      *
* 9.安装邮件服务并配置邮件    20.重启系统                  *
* 10.更改SSH端口号            21.关机                      *
* 11.修改网卡名               22.退出                      *
************************************************************

请选择相应的编号(1-21): 12
请输入IP地址:172.31.0.101
IP 172.31.0.101  available!
请输入子网掩码位数:21
请输入网关地址:172.31.0.2
IP 172.31.0.2  available!
Rocky 8.5 IP地址和网关地址已修改成功,请重新启动系统后生效!

************************************************************
*                      初始化脚本菜单                      *
* 1.禁用SELinux               12.修改IP地址和网关地址      *
* 2.关闭防火墙                13.设置主机名                *
* 3.优化SSH                   14.设置PS1和系统环境变量     *
* 4.设置系统别名              15.禁用SWAP                  *
* 5.1-4全设置                 16.优化内核参数              *
* 6.设置vimrc配置文件         17.优化资源限制参数          *
* 7.设置软件包仓库            18.Ubuntu设置root用户登录    *
* 8.Minimal安装建议安装软件   19.Ubuntu卸载无用软件包      *
* 9.安装邮件服务并配置邮件    20.重启系统                  *
* 10.更改SSH端口号            21.关机                      *
* 11.修改网卡名               22.退出                      *
************************************************************

请选择相应的编号(1-21): 22

#ubuntu系统设置，需要先允许root用户登录
[C:\~]$ ssh raymond@172.31.7.3


Connecting to 172.31.7.3:22...
Connection established.
To escape to local shell, press 'Ctrl+Alt+]'.

Welcome to Ubuntu 18.04.6 LTS (GNU/Linux 4.15.0-156-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Dec 27 13:56:42 CST 2021

  System load:  0.17              Processes:            193
  Usage of /:   2.1% of 91.17GB   Users logged in:      1
  Memory usage: 10%               IP address for ens33: 172.31.7.3
  Swap usage:   0%

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

19 updates can be applied immediately.
18 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

New release '20.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Mon Dec 27 13:56:31 2021
/usr/bin/xauth:  file /home/raymond/.Xauthority does not exist
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

raymond@ubuntu1804:~$ bash reset.sh 

************************************************************
*                      初始化脚本菜单                      *
* 1.禁用SELinux               12.修改IP地址和网关地址      *
* 2.关闭防火墙                13.设置主机名                *
* 3.优化SSH                   14.设置PS1和系统环境变量     *
* 4.设置系统别名              15.禁用SWAP                  *
* 5.1-4全设置                 16.优化内核参数              *
* 6.设置vimrc配置文件         17.优化资源限制参数          *
* 7.设置软件包仓库            18.Ubuntu设置root用户登录    *
* 8.Minimal安装建议安装软件   19.Ubuntu卸载无用软件包      *
* 9.安装邮件服务并配置邮件    20.重启系统                  *
* 10.更改SSH端口号            21.关机                      *
* 11.修改网卡名               22.退出                      *
************************************************************

请选择相应的编号(1-21): 18
请输入密码: 123456
[sudo] password for raymond: Enter new UNIX password: Retype new UNIX password: passwd: password updated successfully
Ubuntu 18.04 root用户登录已设置完成,请重新登录后生效!

************************************************************
*                      初始化脚本菜单                      *
* 1.禁用SELinux               12.修改IP地址和网关地址      *
* 2.关闭防火墙                13.设置主机名                *
* 3.优化SSH                   14.设置PS1和系统环境变量     *
* 4.设置系统别名              15.禁用SWAP                  *
* 5.1-4全设置                 16.优化内核参数              *
* 6.设置vimrc配置文件         17.优化资源限制参数          *
* 7.设置软件包仓库            18.Ubuntu设置root用户登录    *
* 8.Minimal安装建议安装软件   19.Ubuntu卸载无用软件包      *
* 9.安装邮件服务并配置邮件    20.重启系统                  *
* 10.更改SSH端口号            21.关机                      *
* 11.修改网卡名               22.退出                      *
************************************************************

请选择相应的编号(1-21): 22
raymond@ubuntu1804:~$ exit
logout

Connection closed.

Disconnected from remote host(172.31.7.3:22) at 13:57:16.

Type `help' to learn how to use Xshell prompt.

[C:\~]$ ssh root@172.31.7.3


Connecting to 172.31.7.3:22...
Connection established.
To escape to local shell, press 'Ctrl+Alt+]'.

Welcome to Ubuntu 18.04.6 LTS (GNU/Linux 4.15.0-156-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Dec 27 13:57:47 CST 2021

  System load:  0.06              Processes:            199
  Usage of /:   2.1% of 91.17GB   Users logged in:      1
  Memory usage: 11%               IP address for ens33: 172.31.7.3
  Swap usage:   0%

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

19 updates can be applied immediately.
18 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

New release '20.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

/usr/bin/xauth:  file /root/.Xauthority does not exist
root@ubuntu1804:~# mv /home/raymond/reset.sh .
root@ubuntu1804:~# bash reset.sh 

************************************************************
*                      初始化脚本菜单                      *
* 1.禁用SELinux               12.修改IP地址和网关地址      *
* 2.关闭防火墙                13.设置主机名                *
* 3.优化SSH                   14.设置PS1和系统环境变量     *
* 4.设置系统别名              15.禁用SWAP                  *
* 5.1-4全设置                 16.优化内核参数              *
* 6.设置vimrc配置文件         17.优化资源限制参数          *
* 7.设置软件包仓库            18.Ubuntu设置root用户登录    *
* 8.Minimal安装建议安装软件   19.Ubuntu卸载无用软件包      *
* 9.安装邮件服务并配置邮件    20.重启系统                  *
* 10.更改SSH端口号            21.关机                      *
* 11.修改网卡名               22.退出                      *
************************************************************

请选择相应的编号(1-21): 11
Ubuntu 18.04 网卡名已修改成功，请重新启动系统后才能生效!

************************************************************
*                      初始化脚本菜单                      *
* 1.禁用SELinux               12.修改IP地址和网关地址      *
* 2.关闭防火墙                13.设置主机名                *
* 3.优化SSH                   14.设置PS1和系统环境变量     *
* 4.设置系统别名              15.禁用SWAP                  *
* 5.1-4全设置                 16.优化内核参数              *
* 6.设置vimrc配置文件         17.优化资源限制参数          *
* 7.设置软件包仓库            18.Ubuntu设置root用户登录    *
* 8.Minimal安装建议安装软件   19.Ubuntu卸载无用软件包      *
* 9.安装邮件服务并配置邮件    20.重启系统                  *
* 10.更改SSH端口号            21.关机                      *
* 11.修改网卡名               22.退出                      *
************************************************************

请选择相应的编号(1-21): 12
请输入IP地址:172.31.0.103
IP 172.31.0.103  available!
请输入子网掩码位数:21
请输入网关地址:172.31.0.2
IP 172.31.0.2  available!
Ubuntu 18.04 IP地址和网关地址已修改成功,请重新启动系统后生效!

************************************************************
*                      初始化脚本菜单                      *
* 1.禁用SELinux               12.修改IP地址和网关地址      *
* 2.关闭防火墙                13.设置主机名                *
* 3.优化SSH                   14.设置PS1和系统环境变量     *
* 4.设置系统别名              15.禁用SWAP                  *
* 5.1-4全设置                 16.优化内核参数              *
* 6.设置vimrc配置文件         17.优化资源限制参数          *
* 7.设置软件包仓库            18.Ubuntu设置root用户登录    *
* 8.Minimal安装建议安装软件   19.Ubuntu卸载无用软件包      *
* 9.安装邮件服务并配置邮件    20.重启系统                  *
* 10.更改SSH端口号            21.关机                      *
* 11.修改网卡名               22.退出                      *
************************************************************

请选择相应的编号(1-21): 22
```

# 4.实现基于key验证的脚本

```bash
#下面的IP根据自己的k8s集群主机规划设置
[root@ansible-server ansible]# cat ssh_key.sh 
#!/bin/bash
#
#**********************************************************************************************
#Author:        Raymond
#QQ:            88563128
#Date:          2021-12-20
#FileName:      ssh_key.sh
#URL:           raymond.blog.csdn.net
#Description:   ssh_key for CentOS 7/8 & Ubuntu 18.04/24.04 & Rocky 8
#Copyright (C): 2021 All rights reserved
#*********************************************************************************************
COLOR="echo -e \\033[01;31m"
END='\033[0m'

NET_NAME=`ip addr |awk -F"[: ]" '/^2: e.*/{print $3}'`
IP=`ip addr show ${NET_NAME}| awk -F" +|/" '/global/{print $3}'`
export SSHPASS=123456
HOSTS="
172.31.3.101
172.31.3.102
172.31.3.103
172.31.3.104
172.31.3.105
172.31.3.106
172.31.3.107
172.31.3.108
172.31.3.109
172.31.3.110
172.31.3.111
172.31.3.112
172.31.3.113"

os(){
    OS_ID=`sed -rn '/^NAME=/s@.*="([[:alpha:]]+).*"$@\1@p' /etc/os-release`
}

ssh_key_push(){
    rm -f ~/.ssh/id_rsa*
    ssh-keygen -f /root/.ssh/id_rsa -P '' &> /dev/null
    if [ ${OS_ID} == "CentOS" -o ${OS_ID} == "Rocky" ] &> /dev/null;then
        rpm -q sshpass &> /dev/null || { ${COLOR}"安装sshpass软件包"${END};yum -y install sshpass &> /dev/null; }
    else
        dpkg -S sshpass &> /dev/null || { ${COLOR}"安装sshpass软件包"${END};apt -y install sshpass &> /dev/null; }
    fi
    sshpass -e ssh-copy-id -o StrictHostKeyChecking=no ${IP} &> /dev/null
    [ $? -eq 0 ] && echo ${IP} is finished || echo ${IP} is false

    for i in ${HOSTS};do
        sshpass -e scp -o StrictHostKeyChecking=no -r /root/.ssh root@${i}: &> /dev/null
        [ $? -eq 0 ] && echo ${i} is finished || echo ${i} is false
    done

    for i in ${HOSTS};do
        scp /root/.ssh/known_hosts ${i}:.ssh/ &> /dev/null
        [ $? -eq 0 ] && echo ${i} is finished || echo ${i} is false
    done
}

main(){
    os
    ssh_key_push
}

main

[root@ansible-server ansible]# bash ssh_key.sh 
172.31.3.100 is finished
172.31.3.101 is finished
172.31.3.102 is finished
172.31.3.103 is finished
172.31.3.104 is finished
172.31.3.105 is finished
172.31.3.106 is finished
172.31.3.107 is finished
172.31.3.108 is finished
172.31.3.109 is finished
172.31.3.110 is finished
172.31.3.111 is finished
172.31.3.112 is finished
172.31.3.113 is finished
172.31.3.101 is finished
172.31.3.102 is finished
172.31.3.103 is finished
172.31.3.104 is finished
172.31.3.105 is finished
172.31.3.106 is finished
172.31.3.107 is finished
172.31.3.108 is finished
172.31.3.109 is finished
172.31.3.110 is finished
172.31.3.111 is finished
172.31.3.112 is finished
172.31.3.113 is finished
```

# 5.系统初始化和安装软件包

## 5.1 系统初始化

```bash
[root@ansible-server ansible]# cd roles/reset/
[root@ansible-server reset]# ls
tasks  templates  vars

#下面VIP设置成自己的keepalived里的VIP(虚拟IP)地址，HARBOR_DOMAIN的地址设置成自己的harbor域名地址，ROCKY_URL是Rocky系统的镜像仓库，URL是CentOS和Ubuntu镜像仓库的地址，根据自己情况修改
[root@ansible-server reset]# vim vars/main.yml
VIP: 172.31.3.188
HARBOR_DOMAIN: harbor.raymonds.cc
ROCKY_URL: mirrors.ustc.edu.cn
URL: mirrors.cloud.tencent.com

#下面就是每个阶段的执行命令，后面不再说明
[root@ansible-server ansible]# ansible-playbook reset_role.yml
```

## 5.2 安装软件包

```bash
#由于同时多台主机安装软件会有软件超时，把reset阶段分开，因为上面set_hosts.yml需要所有主机同时执行剧本，才能写进去所有主机的host。

[root@ansible-server ansible]# vim reset_installpackage_role.yml 
---
- hosts: all
  serial: 3 #后续如果有同时多台主机安装软件软件超时，也可以加上这行内容，经测试某些情况下多台主机安装软件，超过3台就会报错，所以这里写了3台

  roles:
    - role: reset-installpackage

[root@ansible-server ansible]# ansible-playbook reset_installpackage_role.yml 
```

# 6.chrony

## 6.1 chrony-server

```bash
#安装时间服务器服务端
[root@ansible-server ansible]# ansible-playbook chrony_server_role.yml
```

## 6.2 chrony-client

```bash
#安装时间服务器客户端
[root@ansible-server ansible]# cd roles/chrony-client/
[root@ansible-server chrony-client]# ls
handlers  tasks  vars

#下面IP设置成chrony-server的IP地址，SERVER1设置ha1的IP地址，SERVER2设置ha2的IP地址
[root@ansible-server chrony-client]# vim vars/main.yml
SERVER1: 172.31.3.104
SERVER2: 172.31.3.105

[root@ansible-server ansible]# ansible-playbook chrony_client_role.yml
```

# 7.haproxy

```bash
[root@ansible-server ansible]# cd roles/haproxy/
[root@ansible-server haproxy]# ls
files  tasks  templates  vars

#lua包和haproxy源码包根据自己需求去更换到相应目录
[root@ansible-server haproxy]# wget http://www.lua.org/ftp/lua-5.4.3.tar.gz -P files/
[root@ansible-server haproxy]# wget https://www.haproxy.org/download/2.4/src/haproxy-2.4.10.tar.gz -P files/

[root@ansible-server haproxy]# vim vars/main.yml
SRC_DIR: /usr/local/src #源码解压目录
LUA_FILE: lua-5.4.3.tar.gz #lua包名
HAPROXY_FILE: haproxy-2.4.10.tar.gz #haproxy包名
HAPROXY_INSTALL_DIR: /apps/haproxy #haproxy安装目录
STATS_AUTH_USER: admin	#状态页登录用户
STATS_AUTH_PASSWORD: 123456	#状态页密码
VIP: 172.31.3.188 #IP设置成自己的keepalived里的VIP(虚拟IP)地址

[root@ansible-server ansible]# ansible-playbook haproxy_role.yml
```

# 8.keepalived

## 8.1 keepalived-master

```bash
[root@ansible-server ansible]# cd roles/keepalived-master/
[root@ansible-server keepalived-master]# ls
files  tasks  templates  vars

#keepalived源码包根据自己需求去更换到相应目录
[root@ansible-server keepalived-master]#  wget https://keepalived.org/software/keepalived-2.2.4.tar.gz -P files/

[root@ansible-server keepalived-master]# vim vars/main.yml
URL: mirrors.cloud.tencent.com #URL是CentOS和Ubuntu镜像仓库的地址，根据自己情况修改
ROCKY_URL: mirrors.sjtug.sjtu.edu.cn #ROCKY_URL是Rocky系统的镜像仓库
KEEPALIVED_FILE: keepalived-2.2.4.tar.gz #keepalived包名
SRC_DIR: /usr/local/src #源码解压目录
KEEPALIVED_INSTALL_DIR: /apps/keepalived #keepalived安装目录
STATE: MASTER #master服务器
PRIORITY: 100 #优先级
VIP: 172.31.3.188 #VIP设置成自己的keepalived里的VIP(虚拟IP)地址

[root@ansible-server ansible]# ansible-playbook keepalived_master_role.yml 
```

## 8.2 keepalived-backup

```bash
[root@ansible-server ansible]# cd roles/keepalived-backup/
[root@ansible-server keepalived-master]# ls
files  tasks  templates  vars

#keepalived源码包根据自己需求去更换到相应目录
[root@ansible-server keepalived-backup]#  wget https://keepalived.org/software/keepalived-2.2.4.tar.gz -P files/

[root@ansible-server keepalived-backup]# vim vars/main.yml
URL: mirrors.cloud.tencent.com #URL是CentOS和Ubuntu镜像仓库的地址，根据自己情况修改
ROCKY_URL: mirrors.sjtug.sjtu.edu.cn #ROCKY_URL是Rocky系统的镜像仓库
KEEPALIVED_FILE: keepalived-2.2.4.tar.gz #keepalived包名
SRC_DIR: /usr/local/src #源码解压目录
KEEPALIVED_INSTALL_DIR: /apps/keepalived #keepalived安装目录
STATE: BACKUP #backup服务器
PRIORITY: 90 #优先级
VIP: 172.31.3.188 #VIP设置成自己的keepalived里的VIP(虚拟IP)地址

[root@ansible-server ansible]# ansible-playbook keepalived_backup_role.yml 
```

# 9.harbor

## 9.1 docker基于二进制包  

```bash
[root@ansible-server ansible]# cd roles/docker-binary/
[root@ansible-server docker-binary]# ls
files  tasks  vars  templates

#docker二进制包文件，根据自己情况替换到相应目录
[root@ansible-server docker-binary]# wget https://mirrors.cloud.tencent.com/docker-ce/linux/static/stable/x86_64/docker-19.03.9.tgz -P files/

[root@ansible-server docker-binary]# vim vars/main.yml
DOCKER_VERSION: 19.03.9 #docker版本
HARBOR_DOMAIN: harbor.raymonds.cc #HARBOR_DOMAIN的地址设置成自己的harbor域名地址
```

## 9.2 docker-compose

```bash
[root@ansible-server ansible]# cd roles/docker-compose/
[root@ansible-server docker-compose]# ls
files  tasks

#docker-compose文件根据自己需求去更换到相应目录
[root@ansible-server docker-compose]# wget https://github.com/docker/compose/releases/download/1.29.2/docker-compose-Linux-x86_64 -P files
```

## 9.3 harbor

```bash
[root@ansible-server ansible]# cd roles/harbor/
[root@ansible-server harbor]# ls
files  meta  tasks  templates  vars

#harbor二进制包文件根据自己需求去更换到相应目录
[root@ansible-server harbor]# wget https://github.com/goharbor/harbor/releases/download/v2.4.1/harbor-offline-installer-v2.4.1.tgz -P files/

[root@ansible-server harbor]# vim vars/main.yml
HARBOR_INSTALL_DIR: /apps #harbor安装目录
HARBOR_VERSION: 2.4.1	#harbor版本
HARBOR_ADMIN_PASSWORD: 123456 #harbor仓库登录密码

[root@ansible-server ansible]# ansible-playbook harbor_role.yml
```

## 9.4 创建harbor仓库

这步一定要做，不然后面镜像下载了上传不到harbor，ansible会执行出错

在harbor01新建项目google_containers

![](images/001.jpg)



![](images/002.jpg)



在harbor02新建项目google_containers

![](images/003.jpg)



![](images/004.jpg)



在harbor02上新建目标

![](images/005.jpg)



![](images/006.jpg)



在harbor02上新建规则

![](images/007.jpg)



![](images/008.jpg)



在harbor01上新建目标

![](images/009.jpg)



![](images/010.jpg)



在harbor01上新建规则

![](images/011.jpg)



![](images/012.jpg)

# 10.部署etcd

## 10.1 安装etcd

```bash
[root@ansible-server ansible]# mkdir -p roles/etcd/{tasks,files,vars,templates}
[root@ansible-server ansible]# cd roles/etcd/
[root@ansible-server etcd]# ls
files  tasks  templates  vars

#etcd文件根据自己版本去替换到相应目录
[root@ansible-server etcd]# wget https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-amd64.tar.gz
[root@ansible-server etcd]# mkdir files/etcd
[root@ansible-server etcd]# tar -xf etcd-v3.4.13-linux-amd64.tar.gz --strip-components=1 -C files/etcd/ etcd-v3.4.13-linux-amd64/etcd{,ctl}
[root@ansible-server etcd]# ls files/etcd/
etcd  etcdctl
[root@ansible-server etcd]# rm -f etcd-v3.4.13-linux-amd64.tar.gz

#颁发证书文件
[root@ansible-server etcd]# wget "https://pkg.cfssl.org/R1.2/cfssl_linux-amd64" -O files/cfssl
[root@ansible-server etcd]# wget "https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64" -O files/cfssljson

#下面ETCD02和ETCD03的IP地址根据自己的更改
[root@ansible-server etcd]# vim vars/main.yml
ETCD_CLUSTER: etcd #etcd集群名称
K8S_CLUSTER: kubernetes	#k8s集群名称
ETCD_CERT: #etcd证书文件
  - etcd-ca-key.pem
  - etcd-ca.pem
  - etcd-key.pem
  - etcd.pem

ETCD02: 172.31.3.109
ETCD03: 172.31.3.110

[root@ansible-server ansible]# ansible-playbook etcd_role.yml
```

## 10.2 验证etcd

```bash
[root@k8s-etcd01 ~]# export ETCDCTL_API=3

[root@k8s-etcd01 ~]# etcdctl --endpoints="172.31.3.108:2379,172.31.3.109:2379,172.31.3.110:2379" --cacert=/etc/kubernetes/pki/etcd/etcd-ca.pem --cert=/etc/kubernetes/pki/etcd/etcd.pem --key=/etc/kubernetes/pki/etcd/etcd-key.pem  endpoint status --write-out=table
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|     ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 172.31.3.108:2379 | a9fef56ff96ed75c |  3.4.13 |   20 kB |     false |      false |         2 |          8 |                  8 |        |
| 172.31.3.109:2379 | 8319ef09e8b3d277 |  3.4.13 |   20 kB |     false |      false |         2 |          8 |                  8 |        |
| 172.31.3.110:2379 | 209a1f57c506dba2 |  3.4.13 |   33 kB |      true |      false |         2 |          8 |                  8 |        |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

# 11.部署Containerd

```bash
[root@ansible-server ansible]# cd roles/containerd-binary/
[root@ansible-server containerd-binary]# ls
files  tasks  vars

[root@ansible-server containerd-binary]# vim vars/main.yml
DOCKER_VERSION: 20.10.14 #docker版本
HARBOR_DOMAIN: harbor.raymonds.cc #HARBOR_DOMAIN的地址设置成自己的harbor域名地址
USERNAME: admin #harbor用户名
PASSWORD: 123456 #harbor登录密码

[root@ansible-server ansible]# ansible-playbook containerd_binary_role.yml
```

# 12.部署master

## 12.1 安装master组件

```bash
[root@ansible-server ansible]# cd roles/kubernetes-master/
[root@ansible-server kubernetes-master]# ls
files  tasks  templates  vars

#下面MASTER01、MASTER02和MASTER03的IP地址根据自己的更改
[root@ansible-server kubernetes-master]# vim vars/main.yml
ETCD_CERT: #etcd证书文件
  - etcd-ca-key.pem
  - etcd-ca.pem
  - etcd-key.pem
  - etcd.pem

MASTER01: 172.31.3.101
MASTER02: 172.31.3.102
MASTER03: 172.31.3.103

#kubernetes二进制包文件
[root@ansible-server kubernetes-master]# wget https://dl.k8s.io/v1.20.14/kubernetes-server-linux-amd64.tar.gz
[root@ansible-server kubernetes-master]# mkdir files/bin
[root@ansible-server kubernetes-master]# tar -xf kubernetes-server-linux-amd64.tar.gz  --strip-components=3 -C files/bin kubernetes/server/bin/kube{let,ctl,-apiserver,-controller-manager,-scheduler,-proxy}
[root@ansible-server kubernetes-master]# ls files/bin/
kube-apiserver  kube-controller-manager  kubectl  kubelet  kube-proxy  kube-scheduler
[root@ansible-server kubernetes-master]# rm -f kubernetes-server-linux-amd64.tar.gz

#证书颁发工具文件
[root@ansible-server kubernetes-master]# wget "https://pkg.cfssl.org/R1.2/cfssl_linux-amd64" -O files/cfssl
[root@ansible-server kubernetes-master]# wget "https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64" -O files/cfssljson
[root@ansible-server kubernetes-master]# ls files/
bin  cfssl  cfssljson

[root@ansible-server kubernetes-master]# vim vars/main.yml
...
SERVICE_IP: 10.96.0.1  #SERVICE_IP变量改成自己规划的service_ip地址
VIP: 172.31.3.188 #VIP设置成自己的keepalived里的VIP(虚拟IP)地址
K8S_CLUSTER: kubernetes #k8s集群名称

KUBERNETES_CERT: #master需要证书文件
  - ca.csr
  - ca-key.pem
  - ca.pem
  - apiserver.csr
  - apiserver-key.pem
  - apiserver.pem
  - front-proxy-ca.csr
  - front-proxy-ca-key.pem
  - front-proxy-ca.pem
  - front-proxy-client.csr
  - front-proxy-client-key.pem
  - front-proxy-client.pem
  - controller-manager.csr
  - controller-manager-key.pem
  - controller-manager.pem
  - scheduler.csr
  - scheduler-key.pem
  - scheduler.pem
  - admin.csr
  - admin-key.pem
  - admin.pem
  - sa.key
  - sa.pub

KUBECONFIG: #配置文件
  - controller-manager.kubeconfig
  - scheduler.kubeconfig
  - admin.kubeconfig

[root@ansible-server kubernetes-master]# vim vars/main.yml
...
KUBE_DIRECTROY: #k8s需要目录
  - /etc/kubernetes/manifests/
  - /etc/systemd/system/kubelet.service.d
  - /var/lib/kubelet
  - /var/log/kubernetes

SERVICE_SUBNET: 10.96.0.0/12 #SERVICE_SUBNET改成自己规划的service网段地址
POD_SUBNET: 192.168.0.0/12 #POD_SUBNET改成自己规划的容器网段

MASTER: #MASTER变量ip改成master02和master03的IP
  - 172.31.3.102
  - 172.31.3.103

CLUSTERDNS: 10.96.0.10 #CLUSTERDNS改成service网段的第10个IP
PKI_DIR: /etc/kubernetes/pki #pki文件创建目录
K8S_DIR: /etc/kubernetes #k8s安装目录

[root@ansible-server ansible]# ansible-playbook kubernetes_master_role.yml
```

## 12.2 验证master

```bash
[root@k8s-master01 ~]# kubectl get nodes
NAME                         STATUS     ROLES    AGE     VERSION
k8s-master01.example.local   NotReady   <none>   3m38s   v1.23.6
k8s-master02.example.local   NotReady   <none>   3m37s   v1.23.6
k8s-master03.example.local   NotReady   <none>   3m37s   v1.23.6
```

# 13.部署node

## 13.1 安装node组件

```bash
[root@ansible-server ansible]# cd roles/kubernetes-node/
[root@ansible-server kubernetes-node]# ls
files  tasks  templates  vars

[root@ansible-server kubernetes-node]# vim vars/main.yml
ETCD_CERT: #etcd证书文件
  - etcd-ca-key.pem
  - etcd-ca.pem
  - etcd-key.pem
  - etcd.pem

#下面NODE01、NODE02和NODE03的IP地址根据自己的更改
NODE01: 172.31.3.111
NODE02: 172.31.3.112
NODE03: 172.31.3.113

[root@ansible-server kubernetes-node]# vim vars/main.yml
...
NODE: #node节点IP地址
  - 172.31.3.111
  - 172.31.3.112
  - 172.31.3.113

[root@ansible-server kubernetes-node]# vim vars/main.yml
...
KUBE_DIRECTROY: #node节点目录
  - /etc/kubernetes/manifests/
  - /etc/systemd/system/kubelet.service.d
  - /var/lib/kubelet
  - /var/log/kubernetes
                                
CLUSTERDNS: 10.96.0.10 #CLUSTERDNS改成service网段的第10个IP
POD_SUBNET: 192.168.0.0/12 #POD_SUBNET改成自己规划的容器网段

[root@ansible-server ansible]# ansible-playbook kubernetes_node_role.yml
```

## 13.2 验证node

```bash
[root@k8s-master01 ~]# kubectl get nodes
NAME                         STATUS     ROLES    AGE   VERSION
k8s-master01.example.local   NotReady   <none>   12m   v1.23.6
k8s-master02.example.local   NotReady   <none>   12m   v1.23.6
k8s-master03.example.local   NotReady   <none>   12m   v1.23.6
k8s-node01.example.local     NotReady   <none>   5s    v1.23.6
k8s-node02.example.local     NotReady   <none>   5s    v1.23.6
k8s-node03.example.local     NotReady   <none>   5s    v1.23.6
```

# 14.安装Calico

## 14.1 安装calico

```bash
[root@ansible-server ansible]# cd roles/calico
[root@ansible-server calico]# ls
tasks  templates  vars

[root@ansible-server calico]# vim vars/main.yml
HARBOR_DOMAIN: harbor.raymonds.cc #HARBOR_DOMAIN的地址设置成自己的harbor域名地址
USERNAME: admin #harbor用户名
PASSWORD: 123456 #harbor登录密码
POD_SUBNET: 192.168.0.0/12 #POD_SUBNET改成自己规划的容器网段

[root@ansible-server ansible]# ansible-playbook calico_role.yml 
```

## 14.2 验证calico

```bash
[root@k8s-master01 ~]# kubectl get pod -n kube-system |grep calico
calico-kube-controllers-786596c789-gkkfk   1/1     Running   0          77s
calico-node-4d5hz                          1/1     Running   0          77s
calico-node-4v697                          1/1     Running   0          77s
calico-node-g5nk9                          1/1     Running   0          77s
calico-node-ljvlp                          1/1     Running   0          77s
calico-node-vttcj                          1/1     Running   0          77s
calico-node-wwhrl                          1/1     Running   0          77s

[root@k8s-master01 ~]# kubectl get nodes
NAME                         STATUS   ROLES    AGE   VERSION
k8s-master01.example.local   Ready    <none>   39m   v1.23.6
k8s-master02.example.local   Ready    <none>   39m   v1.23.6
k8s-master03.example.local   Ready    <none>   39m   v1.23.6
k8s-node01.example.local     Ready    <none>   27m   v1.23.6
k8s-node02.example.local     Ready    <none>   27m   v1.23.6
k8s-node03.example.local     Ready    <none>   27m   v1.23.6
```

# 15.安装 CoreDNS

## 15.1 安装 CoreDNS

```bash
[root@ansible-server ansible]# cd roles/coredns/
[root@ansible-server coredns]# ls
tasks  templates  vars

[root@ansible-server coredns]# vim vars/main.yml
CLUSTERDNS: 10.96.0.10 #CLUSTERDNS改成自己规划的service网段的第10个IP地址
HARBOR_DOMAIN: harbor.raymonds.cc #HARBOR_DOMAIN的地址设置成自己的harbor域名地址
USERNAME: admin #harbor用户名
PASSWORD: 123456 #harbor登录密码

[root@ansible-server ansible]# ansible-playbook coredns_role.yml
```

## 15.2 验证coredns

```bash
[root@k8s-master01 ~]# kubectl get po -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-847c895554-cncc4   1/1     Running   0          18s
```

# 16.安装Metrics

## 16.1 安装metrics

```bash
[root@ansible-server ansible]# cd roles/metrics/
[root@ansible-server metrics]# ls
files  tasks  vars

#下面HARBOR_DOMAIN的地址设置成自己的harbor域名地址
[root@ansible-server metrics]# vim vars/main.yml
HARBOR_DOMAIN: harbor.raymonds.cc
USERNAME: admin #harbor用户名
PASSWORD: 123456 #harbor登录密码

[root@ansible-server ansible]# ansible-playbook metrics_role.yml
```

## 16.2 验证metrics

```bash
[root@k8s-master01 ~]# kubectl get pod -A |grep metrics
kube-system   metrics-server-58bcbdcdcd-l7twx           1/1     Running   0          18s

[root@k8s-master01 ~]# kubectl top node
NAME                         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master01.example.local   506m         25%    1484Mi          38%       
k8s-master02.example.local   142m         7%     1381Mi          36%       
k8s-master03.example.local   247m         12%    1389Mi          36%       
k8s-node01.example.local     79m          3%     721Mi           18%       
k8s-node02.example.local     80m          4%     726Mi           19%       
k8s-node03.example.local     90m          4%     784Mi           20%  
```

# 17.安装dashboard

## 17.1 安装dashboard

```bash
[root@ansible-server ansible]# cd roles/dashboard/
[root@ansible-server dashboard]# ls
files  tasks  templates  vars

[root@ansible-server dashboard]# vim vars/main.yml
HARBOR_DOMAIN: harbor.raymonds.cc #HARBOR_DOMAIN的地址设置成自己的harbor域名地址
USERNAME: admin #harbor用户名
PASSWORD: 123456 #harbor登录密码
NODEPORT: 30005 #dashboard暴露端口

[root@ansible-server ansible]# ansible-playbook dashboard_role.yml
```

## 17.2 登录dashboard

https://172.31.3.101:30005

![](images/013.jpg)

查看token值：

```bash
[root@k8s-master01 ~]# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-gv2wt
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 6a9ac08f-672c-4c51-81a1-d1a48539aa1e

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1411 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IkFpTHJHZUppeEV4MTMzUXVtNFBFNm9Vck5TOGhiaWo5a0pyVnFIRWlrcUEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWd2Mnd0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI2YTlhYzA4Zi02NzJjLTRjNTEtODFhMS1kMWE0ODUzOWFhMWUiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.oTSKkV7x9Nlaoe-TJni1uKyax2-FwDv47CuIzQbl5tmibmcEJ1vmio9V5eY4kwwoWlDOVYj4MVQ192OWyMVyoNkYOEv8_ZvxqK1xxpicPDDz64-qUYKKI4PbGDMsO40djeiJR-8PKM1D4LmxBsaBR-QjPq9rwG6iKJ2poOr-GTgoODTCCn7DTShEuKNtPQxDf7rGP3Ofccc-C1jBcd7M07am-6qtWsg8GFEwhbk4HXTZLyThQGe99Tth74OxQFu1izUzbN_JnE8HM9jjf1sqgFeAXVoxx6hitTKah8KQplfta5OCLEhxc9Q82rpeD7RjNUyrc1AN7CfLcXS-dKvJfw
```



![](images/014.jpg)


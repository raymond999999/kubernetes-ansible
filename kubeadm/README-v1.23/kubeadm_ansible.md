# kubeadm ansible 安装方法

[TOC]



# 1.高可用Kubernetes集群规划

集群规划根据自己的情况设置，下面标出了使用需要修改的地方，加以标注，具体内容参考博客内容

| 角色    | 机器名                       | 机器配置 | ip地址                         | 安装软件                                          |
| ------- | ---------------------------- | -------- | ------------------------------ | ------------------------------------------------- |
| ansible | ansible-server.example.local | 2C2G     | 172.31.3.100                   | ansible                                           |
| master1 | k8s-master01.example.local   | 2C4G     | 172.31.3.101                   | chrony-client、docker、kubeadm 、kubelet、kubectl |
| master2 | k8s-master02.example.local   | 2C4G     | 172.31.3.102                   | chrony-client、docker、kubeadm 、kubelet、kubectl |
| master3 | k8s-master03.example.local   | 2C4G     | 172.31.3.103                   | chrony-client、docker、kubeadm 、kubelet、kubectl |
| ha1     | k8s-ha01.example.local       | 2C2G     | 172.31.3.104 172.31.3.188(vip) | chrony-server、haproxy、keepalived                |
| ha2     | k8s-ha02.example.local       | 2C2G     | 172.31.3.105                   | chrony-server、haproxy、keepalived                |
| harbor1 | k8s-harbor01.example.local   | 2C2G     | 172.31.3.106                   | chrony-client、docker、docker-compose、harbor     |
| harbor2 | k8s-harbor02.example.local   | 2C2G     | 172.31.3.107                   | chrony-client、docker、docker-compose、harbor     |
| node1   | k8s-node01.example.local     | 2C4G     | 172.31.3.108                   | chrony-client、docker、kubeadm 、kubelet          |
| node2   | k8s-node02.example.local     | 2C4G     | 172.31.3.109                   | chrony-client、docker、kubeadm 、kubelet          |
| node3   | k8s-node03.example.local     | 2C4G     | 172.31.3.110                   | chrony-client、docker、kubeadm 、kubelet          |

软件版本信息和Pod、Service网段规划：

| **配置信息**       | 备注                                             |
| ------------------ | ------------------------------------------------ |
| 支持的操作系统版本 | CentOS 7.9/stream 8、Rocky 8、Ubuntu 18.04/20.04 |
| Docker版本         | 20.10.14                                         |
| Containerd版本        | 1.15.11                                          |
| kubeadm版本        | 1.23.6                                          |
| Pod网段            | 192.168.0.0/12                                   |
| Service网段        | 10.96.0.0/12                                     |

# 2.安装ansible和配置

## 2.1 安装ansible

```bash
#CentOS
[root@ansible-server ~]# yum -y install ansible

[root@ansible-server ~]# ansible --version
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
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

[node]
172.31.3.108 hname=k8s-node01
172.31.3.109 hname=k8s-node02
172.31.3.110 hname=k8s-node03

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
172.31.3.110"

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
```

# 5.系统初始化和安装软件包

## 5.1 系统初始化roles

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

## 5.2 安装软件包roles

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

# 6.安装chrony

## 6.1 安装chrony-server

```bash
#安装时间服务器服务端
[root@ansible-server ansible]# ansible-playbook chrony_server_role.yml
```

## 6.2 安装chrony-client

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

# 7.安装haproxy

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

# 8.安装keepalived

## 8.1 安装keepalived-master

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

## 8.2 安装keepalived-backup

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

# 9.安装harbor

## 9.1  docker基于镜像仓库  

```bash
[root@ansible-server ansible]# cd roles/docker/
[root@ansible-server docker]# ls
tasks  templates  vars

[root@ansible-server docker]# vim vars/main.yml
DOCKER_VERSION: 19.03.15 #docker版本
DOCKER_MIRRORS: mirrors.cloud.tencent.com #docker镜像仓库
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

# 10.安装containerd

```bash
[root@ansible-server ansible]# cd roles/containerd/
[root@ansible-server containerd]# ls
tasks  vars

[root@ansible-server containerd]# vim vars/main.yml 
DOCKER_VERSION: 20.10.14 #docker版本
DOCKER_MIRRORS: mirrors.cloud.tencent.com #docker镜像仓库地址
HARBOR_DOMAIN: harbor.raymonds.cc #HARBOR_DOMAIN的地址设置成自己的harbor域名地址
USERNAME: admin #harbor登录用户名
PASSWORD: 123456 #harbor登录密码

[root@ansible-server ansible]# ansible-playbook containerd_role.yml
```

# 11.安装kubeadm

```bash
[root@ansible-server ansible]# cd roles/kubeadm/
[root@ansible-server kubeadm]# ls
files  tasks  vars

[root@ansible-server kubeadm]# vim vars/main.yml
KUBEADM_MIRRORS: mirrors.aliyun.com #kubeadm镜像仓库
KUBEADM_VERSION: 1.20.14 #kubeadm版本
HARBOR_DOMAIN: harbor.raymonds.cc #HARBOR_DOMAIN的地址设置成自己的harbor域名地址
USERNAME: admin #harbor登录用户名
PASSWORD: 123456 #harbor登录密码
VIP: 172.31.3.188 #VIP设置成自己的keepalived里的VIP(虚拟IP)地址
POD_SUBNET: 192.168.0.0/12 #POD_SUBNET改成自己规划的容器网段
SERVICE_SUBNET: 10.96.0.0/12 #SERVICE_SUBNET改成自己规划的service网段地址

[root@ansible-server ansible]# ansible-playbook kubeadm_role.yml
```

# 12.集群初始化

```bash
[root@k8s-master01 ~]# kubeadm init --config /root/kubeadm-config.yaml  --upload-certs
[init] Using Kubernetes version: v1.23.6
[preflight] Running pre-flight checks
	[WARNING FileExisting-tc]: tc not found in system path
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master01.example.local kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.example.local] and IPs [10.96.0.1 172.31.3.101 172.31.3.188]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master01.example.local localhost] and IPs [172.31.3.101 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master01.example.local localhost] and IPs [172.31.3.101 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 22.035806 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.23" in namespace kube-system with the configuration for the kubelets in the cluster
NOTE: The "kubelet-config-1.23" naming of the kubelet ConfigMap is deprecated. Once the UnversionedKubeletConfigMap feature gate graduates to Beta the default name will become just "kubelet-config". Kubeadm upgrade will handle this transition transparently.
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
a356a0307e08f530b8b93b5d1b6c3c8d372f8808be369632aa61434996e5928b
[mark-control-plane] Marking the node k8s-master01.example.local as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-master01.example.local as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 7t2weq.bjbawausm0jaxury
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 172.31.3.188:6443 --token 7t2weq.bjbawausm0jaxury \
	--discovery-token-ca-cert-hash sha256:fd3bcda7669c8e047de6d663f12c4a7c662ae9c18bd2edadcb30ab5bf3c0bdcf \
	--control-plane --certificate-key a356a0307e08f530b8b93b5d1b6c3c8d372f8808be369632aa61434996e5928b

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.3.188:6443 --token 7t2weq.bjbawausm0jaxury \
	--discovery-token-ca-cert-hash sha256:fd3bcda7669c8e047de6d663f12c4a7c662ae9c18bd2edadcb30ab5bf3c0bdcf
```

Master01节点配置环境变量，用于访问Kubernetes集群：

```bash
[root@k8s-master01 ~]# cat >> /root/.bashrc <<EOF
export KUBECONFIG=/etc/kubernetes/admin.conf
EOF

[root@k8s-master01 ~]# source .bashrc 

#Kubectl 自动补全
#CentOS
[root@k8s-master01 ~]# yum -y install bash-completion

#Ubuntu
[root@k8s-master01 ~]# apt -y install bash-completion

[root@k8s-master01 ~]# source <(kubectl completion bash)
[root@k8s-master01 ~]# echo "source <(kubectl completion bash)" >> ~/.bashrc 
[root@k8s-master01 ~]# exit
logout
```

查看节点状态：

```bash
[root@k8s-master01 ~]# kubectl get nodes 
NAME                         STATUS     ROLES                  AGE    VERSION
k8s-master01.example.local   NotReady   control-plane,master   2m5s   v1.23.6
```

采用初始化安装方式，所有的系统组件均以容器的方式运行并且在kube-system命名空间内，此时可以查看Pod状态：

```bash
[root@k8s-master01 ~]# kubectl get pods -n kube-system -o wide
NAME                                                 READY   STATUS    RESTARTS   AGE     IP             NODE                         NOMINATED NODE   READINESS GATES
coredns-68db94d5b7-dsb48                             0/1     Pending   0          2m38s   <none>         <none>                       <none>           <none>
coredns-68db94d5b7-xkxcb                             0/1     Pending   0          2m38s   <none>         <none>                       <none>           <none>
etcd-k8s-master01.example.local                      1/1     Running   0          2m33s   172.31.3.101   k8s-master01.example.local   <none>           <none>
kube-apiserver-k8s-master01.example.local            1/1     Running   0          2m33s   172.31.3.101   k8s-master01.example.local   <none>           <none>
kube-controller-manager-k8s-master01.example.local   1/1     Running   0          2m33s   172.31.3.101   k8s-master01.example.local   <none>           <none>
kube-proxy-rx8nz                                     1/1     Running   0          2m38s   172.31.3.101   k8s-master01.example.local   <none>           <none>
kube-scheduler-k8s-master01.example.local            1/1     Running   0          2m33s   172.31.3.101   k8s-master01.example.local   <none>           <none>
```

# 13.高可用Master

添加master02和master03:

```bash
kubeadm join 172.31.3.188:6443 --token 7t2weq.bjbawausm0jaxury \
	--discovery-token-ca-cert-hash sha256:fd3bcda7669c8e047de6d663f12c4a7c662ae9c18bd2edadcb30ab5bf3c0bdcf \
	--control-plane --certificate-key a356a0307e08f530b8b93b5d1b6c3c8d372f8808be369632aa61434996e5928b

[root@k8s-master01 ~]# kubectl get nodes 
NAME                         STATUS     ROLES                  AGE     VERSION
k8s-master01.example.local   NotReady   control-plane,master   4m32s   v1.23.6
k8s-master02.example.local   NotReady   control-plane,master   52s     v1.23.6
k8s-master03.example.local   NotReady   control-plane,master   8s      v1.23.6
```

# 14.Node节点的配置

添加node01、node02、node03:

```bash
kubeadm join 172.31.3.188:6443 --token 7t2weq.bjbawausm0jaxury \
	--discovery-token-ca-cert-hash sha256:fd3bcda7669c8e047de6d663f12c4a7c662ae9c18bd2edadcb30ab5bf3c0bdcf

[root@k8s-master01 ~]# kubectl get nodes 
NAME                         STATUS     ROLES                  AGE     VERSION
k8s-master01.example.local   NotReady   control-plane,master   6m1s    v1.23.6
k8s-master02.example.local   NotReady   control-plane,master   2m21s   v1.23.6
k8s-master03.example.local   NotReady   control-plane,master   97s     v1.23.6
k8s-node01.example.local     NotReady   <none>                 33s     v1.23.6
k8s-node02.example.local     NotReady   <none>                 18s     v1.23.6
k8s-node03.example.local     NotReady   <none>                 5s      v1.23.6
```

# 15.安装Calico

## 15.1 安装calico

```bash
[root@ansible-server ansible]# cd roles/calico
[root@ansible-server calico]# ls
tasks  templates  vars

#下面HARBOR_DOMAIN的地址设置成自己的harbor域名地址
[root@ansible-server calico]# vim vars/main.yml
HARBOR_DOMAIN: harbor.raymonds.cc
USERNAME: admin #harbor登录用户名
PASSWORD: 123456 #harbor登录密码

[root@ansible-server ansible]# ansible-playbook calico_role.yml 
```

## 15.2 验证calico

```bash
[root@k8s-master01 ~]# kubectl get pod -A |grep calico
kube-system   calico-kube-controllers-786596c789-prcz4             1/1     Running   0             40s
kube-system   calico-node-cpxg4                                    1/1     Running   0             41s
kube-system   calico-node-j5n25                                    1/1     Running   0             41s
kube-system   calico-node-pq4xz                                    1/1     Running   0             41s
kube-system   calico-node-sjgk2                                    1/1     Running   0             41s
kube-system   calico-node-xbkc7                                    1/1     Running   0             41s
kube-system   calico-node-zrww5                                    1/1     Running   0             41s

[root@k8s-master01 ~]# kubectl get nodes 
NAME                         STATUS   ROLES                  AGE   VERSION
k8s-master01.example.local   Ready    control-plane,master   22m   v1.23.6
k8s-master02.example.local   Ready    control-plane,master   19m   v1.23.6
k8s-master03.example.local   Ready    control-plane,master   18m   v1.23.6
k8s-node01.example.local     Ready    <none>                 17m   v1.23.6
k8s-node02.example.local     Ready    <none>                 17m   v1.23.6
k8s-node03.example.local     Ready    <none>                 16m   v1.23.6
```

# 16.安装Metrics

## 16.1 安装metrics

```bash
[root@ansible-server ansible]# cd roles/metrics/
[root@ansible-server metrics]# ls
files  tasks  vars

#下面NODE变量改成自己NODE的IP地址，HARBOR_DOMAIN的地址设置成自己的harbor域名地址
[root@ansible-server metrics]# vim vars/main.yml
NODE:
 - 172.31.3.108
 - 172.31.3.109
 - 172.31.3.110

HARBOR_DOMAIN: harbor.raymonds.cc
USERNAME: admin #harbor登录用户名
PASSWORD: 123456 #harbor登录密码

[root@ansible-server ansible]# ansible-playbook metrics_role.yml
```

## 16.2 验证metrics

```bash
[root@k8s-master01 ~]# kubectl get pod -n kube-system |grep metrics
metrics-server-5b7c76b46c-4lmzj                      1/1     Running   0          39s

[root@k8s-master01 ~]# kubectl top nodes
NAME                         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master01.example.local   151m         7%     1557Mi          40%       
k8s-master02.example.local   150m         7%     1426Mi          37%       
k8s-master03.example.local   131m         6%     1279Mi          33%       
k8s-node01.example.local     75m          3%     876Mi           22%       
k8s-node02.example.local     76m          3%     914Mi           23%       
k8s-node03.example.local     83m          4%     966Mi           25% 
```

# 17.安装dashboard

## 17.1 安装dashboard

```bash
[root@ansible-server ansible]# cd roles/dashboard/
[root@ansible-server dashboard]# ls
files  tasks  templates  vars

[root@ansible-server dashboard]# vim vars/main.yml
HARBOR_DOMAIN: harbor.raymonds.cc #HARBOR_DOMAIN的地址设置成自己的harbor域名地址
USERNAME: admin #harbor登录用户名
PASSWORD: 123456 #harbor登录密码
NODEPORT: 30005 #dashboard暴漏的端口

[root@ansible-server ansible]# ansible-playbook dashboard_role.yml
```

## 17.2 登录dashboard

https://172.31.3.101:30005

![](images/013.jpg)

查看token值：

```bash
[root@k8s-master01 ~]# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-8bwlj
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: c65fb340-06d8-40e6-a344-898ff4ef822f

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1066 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImR1LVBSQVY2bnlXaEZqR1hjN1hENW1iWU1vdXI1Z2ZNTUhvWXNSeFg0d0UifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLThid2xqIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJjNjVmYjM0MC0wNmQ4LTQwZTYtYTM0NC04OThmZjRlZjgyMmYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.qwD3Q72dhyyjFJ53iU2-Cs1ubBGZi603Q3z50munzjGh0QdbIthb8-RusnPzHjDZNEWPtWBUyNiI0U2o7tLNaNZp0ySxHZZW7OaACb6hVVYQZBK5bXzDgKfD2ouMRvPEDkTTfFIdN1EOyZSw5ypFakeBT7ZnJCYSwsIbX5wiBc16umkLt_N-uLx-OvlCOxFrw-HmkktgKiDixn2yomcAUpiVgWkfRtoySe8a1B5wauR9vnGBayqPQMCyVzeMkJW3GOXx4HxOLnQ1TvPVJoZb1XAu_hoKBQ2-sye1anQ0KntSfsmGUYlpreeSdG-sC-enrrjFneH5pllHWb6acMxIAA
```



![](images/014.jpg)

# 18.一些必须的配置更改

将Kube-proxy改为ipvs模式，因为在初始化集群的时候注释了ipvs配置，所以需要自行修改一下：

在master01节点执行

```bash
[root@k8s-master01 ~]# curl 127.0.0.1:10249/proxyMode
iptables

[root@k8s-master01 ~]# kubectl edit cm kube-proxy -n kube-system
    mode: "ipvs"
```

更新Kube-Proxy的Pod：

```bash
[root@k8s-master01 ~]# kubectl patch daemonset kube-proxy -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" -n kube-system
daemonset.apps/kube-proxy patched
```

验证Kube-Proxy模式

```bash
[root@k8s-master01 ~]# curl 127.0.0.1:10249/proxyMode
ipvs

[root@k8s-master01 ~]# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.17.0.1:30005 rr
  -> 192.170.21.194:8443          Masq    1      0          0         
TCP  172.31.3.101:30005 rr
  -> 192.170.21.194:8443          Masq    1      0          0         
TCP  192.162.55.64:30005 rr
  -> 192.170.21.194:8443          Masq    1      0          0         
TCP  10.96.0.1:443 rr
  -> 172.31.3.101:6443            Masq    1      0          0         
  -> 172.31.3.102:6443            Masq    1      0          0         
  -> 172.31.3.103:6443            Masq    1      0          0         
TCP  10.96.0.10:53 rr
  -> 192.169.111.129:53           Masq    1      0          0         
  -> 192.170.21.193:53            Masq    1      0          0         
TCP  10.96.0.10:9153 rr
  -> 192.169.111.129:9153         Masq    1      0          0         
  -> 192.170.21.193:9153          Masq    1      0          0         
TCP  10.100.83.15:443 rr
  -> 192.167.195.131:4443         Masq    1      0          0         
TCP  10.102.101.21:8000 rr
  -> 192.169.111.130:8000         Masq    1      0          0         
TCP  10.106.201.206:443 rr
  -> 192.170.21.194:8443          Masq    1      0          0         
TCP  127.0.0.1:30005 rr
  -> 192.170.21.194:8443          Masq    1      0          0         
UDP  10.96.0.10:53 rr
  -> 192.169.111.129:53           Masq    1      0          0         
  -> 192.170.21.193:53            Masq    1      0          0  
```


---
layout: post
title: mac电脑iTerm免密登陆
category: 技术分享
slug: mac-iterm-ssh-login
Tags: [mac, iterm]
---

## 本地生成keychain免输域密码
首先在本地生成一个keychain，存储你的域密码，使用的时候只需要输入本机密码即可

1. 创建一个keychain
````
security add-generic-password -a 'xianchao.sxc' -s 'ssh_login1.cm4' -w '你的域密码' -T /Applications/iTerm.app itermapp.keychain
````

2. 验证下是否生效
````
security find-generic-password -a'xianchao.sxc' -l 'ssh_login1.cm4' -w  itermapp.keychain
````
会在控制台输入你的域密码

<!--break-->

3. 输入安全
本机密码一定不要外泄，一旦泄露你的域密码就会被被人获取了，删除keychain
````
security delete-keychain
security delete-generic-password -a 'xianchao.sxc'
````
另外是可以创建一个不同于本机的密码来访问你的keychain
````
security create-keychain -p '访问密码' itermapp.keychain
````

## 免密登陆跳板机
利用expect实现交互输入访问密码获取token和域密码实现登陆跳板机,gossh脚本内容如下：
````
#!/usr/bin/expect

## demo: expect ./gossh user@ip pwd_for_ssh

# 先设置变量
set ipaddr [lindex $argv 0]
set password [lindex $argv 1]
set encode [lindex $argv 2]
set timeout 30

# 参数检测
if {$argc < 2} {
  send_user "Usage: ./expectf \$ipaddr \$password\n"
  exit
}

# 若需要编码转换
if {$argc >= 3} {
  catch { spawn luit -encoding $encode ssh $ipaddr }
} else {
  catch { spawn ssh $ipaddr }
}

# 等待输入密码的提示符
expect {
    "*password*" {
        send "$password\r"
    }
    -re "Enter passphrase for key" {
        send "$password\r"
    }
    "*password:*" {
        send "$password\r"
    }
    "connecting (yes/no)" {
        send "yes\r"
        exp_continue
    }
    timeout {
        send_user "expect timeout."
        exit
    }
}

interact
````
下面是访问gossh的脚本，可以写到一个文件里，以后直接访问这个文件即可
````
if [ -e /tmp/ssh_connection_login1.cm4.alibaba.org_22_xianchao.sxc.sock ]; then
   ssh xianchao.sxc@login1.cm4.alibaba.org
else
   screen -R pub expect ~/.ssh/gossh xianchao.sxc@login1.cm4.alibaba.org `security find-    generic-password -l 'ssh_login1.cm4' -a 'xianchao.sxc' -w itermapp.keychain`
fi
````
成功登陆到跳板机后，以后再访问跳板机就不需要之前的输入流程了直接登陆到跳板机

## 线上机器免密登陆
1. 生成密钥文件
````
ssh-keygen -t rsa -P ''
````
2. 获取应用的机器列表  autoget [应用名称] > [应用名称].ips
应用名称可以去psp中查询获取

3. 将公钥拷贝到线上机器
````
pgmscp -A -b -f  [应用名称].ips ~/.ssh/id_rsa.pub ~/.ssh
pgm -A -b -p 5 -f [应用名称].ips "cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys"
````

通过读取应用ip文件把跳板机上的id_rsa.pub拷贝到线上机器对应的路径~/.ssh下并对每台线上机器，把id_rsa.pub文件内容追加到authorized_keys文件中

4. 登陆线上机器脚本
上面已经获取了机器的ip文件通过grep 机器关键词来获取机器名称
````
#!/bin/bash

if [ $# -eq 0 ]; then
     echo "./wcgo hostname keyword"
     exit 0
else
    hostname=`grep $1 wangcai.txt`
fi

echo "login to host:$hostname ...."
sleep 2

if [ "$hostname" = "" ]; then
    echo "not found hostname with keyword:$1"
    exit 0
else
    ssh "$hostname"
fi
````
另存个脚本ssg chmod +x ssg
使用ssg keyword 停留1秒后就可以跳转到对应的机器上，如果不想登陆在这1秒可以ctrl+c 取消掉

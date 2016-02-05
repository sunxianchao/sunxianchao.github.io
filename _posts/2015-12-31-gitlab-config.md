---
layout: post
title: gitlab配置记录
category: 工作记录
Tags: [git, gitlab]
---

### 前提
使用git访问远程仓库地址是使用ssh协议，因此需要配置远程访问的公钥和私钥：
````
ssh-keygen -t rsa -C "alibaba-inc"
````
引号的内容随便替换，回车后会让你输入秘钥的文件名默认就是~/.ssh/id_rsa 可以为不同的git仓库创建不同的名字

### 配置ssh key
打开git仓库的后台配置ssh key，将公钥上传
````
cat ~/.ssh/id_rsa.pub
````
<!--break-->

### config配置
配置用户名和邮箱，方便以后更新，避免每次输入用户名
````
git config --global user.name "括羽"
git config --global user.email "xianchao.sxc@alibaba-inc.com"
````

### 测试
````
ssh -T git@gitlab.alibaba-inc.com
Welcome to GitLab, 括羽!
````

### 配置不同的git仓库
很多时候我们还需要访问除了公司以为的git仓库，如github
这时候需要最好配置不同的公钥访问区分，重复第一步但是输入不同的文件名id_rsa_github
上传id_rsa_github.pub到github后台

在~/.ssh/下创建config文件内容如下：
````
Host github #随意
HostName github.com
User sunxianchao@gmail.com #登陆的邮箱
IdentityFile ~/.ssh/id_rsa_github.pub #公钥本地路径

Host aligit
HostName gitlab.alibaba-inc.com
User xianchao.sxc@alibaba-inc.com
IdentityFile ~/.ssh/id_rsa.pub
````

以上仅是记录一下方便以后查阅！

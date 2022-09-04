---
title: 运维开发笔记
date: 2020-08-14 14:33:42
tags:
- devops
categories: devops
---
### ssh免密登录localhost配置
```bash
生成 ssh keys

ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa

将 ssh keys 输出到 authorized_keys中

cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

切换权限

chmod 0600 ~/.ssh/authorized_keys
```
### base64 加密解密
```bash
加密
base64 <<< "string"
解密
base64 -d <<< c3RyaW5nCg==

```
### npm run dev远程访问
修改config下面host为 0.0.0.0
package.json 的 scripts dev里面加上 --disableHostCheck=true
### 查看uwsgi进程
```bash
ps -ef | grep uwsgi
```
### windows 换行符转 unix换行符
```bash
   用 sed 进行字符串替换

   sed -i 's/\r//g' *.sh
   用 dos2unix 包
   yum install dos2unix
   dos2unix *.sh
```
### nohup 守护进程后台执行任务
```bash
nohup python3  test.py  &>>error.log 
```
### docker 容器内无法使用vi vim 等命令
```bash
apt-get update
或者 yum update
然后
apt-get install vim-tiny
or yum install vim
```


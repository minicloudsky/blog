---
title: linux shell 学习
date: 2019-02-23 16:32:18
tags:
---


## 记录下常用的 shell 命令
- pwd 打印工作目录
- ls 列出目录下文件
  ls -a 列出全部文件，包含隐藏文件
  ls -l  长数据串列出，包含文件的属性与权限
- cd 切换目录
cd 绝对路径	切换路径
cd 相对路径	切换路径
cd ~或者cd	回到自己的家目录
cd -	回到上一次所在目录
cd ..	回到当前目录的上一级目录
cd -P	跳转到实际物理路径，而非快捷方式路径
- mkdir 创建目录
  mkdir -p dir 创建多层目录
- rmdir 删除目录
- touch 创建空文件
- cp 复制文件或目录
  cp source target
  cp -r 递归复制整个目录或文件
  \cp  强制覆盖不提示
- rm 移除文件或目录
  ```shell
  -r	递归删除目录中所有内容
  -f	强制执行删除操作，而不提示用于进行确认。
  -v	显示指令的详细执行过程
```
-mv 移动文件与目录或重命名
  1. mv oldNameFile newNameFile	（功能描述：重命名）
  2. mv /temp/movefile /targetFolder	（功能描述：移动文件）
- cat 查看文件内容
```shell
cat filename
cat -n 显示所有行的行号，包括空行
```
- more指令是一个基于VI编辑器的文本过滤器，它以全屏幕的方式按页显示文本文件的内容。more指令中内置了若干快捷键，详见操作说明。
1. 基本语法：
	more 要查看的文件
2. 操作说明
| 操作	|功能说明|
|空白键 (space)|	代表向下翻一页；|
|Enter	|代表向下翻『一行』；|
|q	|代表立刻离开 more ，不再显示该文件内容。|
|Ctrl+f|	向下滚动一屏|
|Ctrl+b	|返回上一屏|
|=|	输出当前行的行号|
|:f	|输出文件名和当前行的行号|
<!-- more -->
- less指令用来分屏查看文件内容，它的功能与more指令类似，但是比more指令更加强大，支持各种显示终端。less指令在显示文件内容时，并不是一次将整个文件加载之后才显示，而是根据显示需要加载内容，对于显示大型文件具有较高的效率。
1）基本语法：
	less 要查看的文件
2）操作说明
操作	功能说明
空白键	向下翻动一页；
[pagedown]	向下翻动一页
[pageup]	向上翻动一页；
/字串	向下搜寻『字串』的功能；n：向下查找；N：向上查找；
?字串	向上搜寻『字串』的功能；n：向上查找；N：向下查找；
q  	离开 less 这个程序；

- echo输出内容到控制台
基本语法：
		echo [选项] [输出内容]
选项： 
  -e：  支持反斜线控制的字符转换
|控制字符 | 	作用 |
|\\  |	输出\本身|
|\n  |	换行符|
|\t  |	制表符，也就是Tab键|

- head用于显示文件的开头部分内容，默认情况下head指令显示文件的前10行内容。
head 文件	      （功能描述：查看文件头10行内容）
head -n 5 文件      （功能描述：查看文件头5行内容，5可以是任意行数）
-n<行数>	指定显示头部内容的行数

```shell
[root@aliyun ~]# head -10 defer.go 
package main

import "fmt"

func main(){
  //  for i:=0;i<5;i++{
//        defer fmt.Printf("%d ",i)
   // }
    // fmt.Println("\n")
    //x:= []int{1,2,3}
```
- tail用于输出文件中尾部的内容，默认情况下tail指令显示文件的后10行内容。
1.基本语法tail
  1. tail  文件 			（功能描述：查看文件头10行内容）
  2. tail  -n 5 文件 		（功能描述：查看文件头5行内容，5可以是任意行数）
  3. tail  -f  文件		（功能描述：实时追踪该文档的所有更新）
2. 选项说明
选项	功能
-n<行数>	输出文件尾部n行内容
-f	显示文件最新追加的内容，监视文件变化
实时追踪该档的所有更新
```shell
[root@aliyun ~]# tail -f a.txt
date~123

```
- 输出重定向和 >> 追加
1）基本语法：ll
（1）ls -l >文件		（功能描述：列表的内容写入文件a.txt中（覆盖写））
（2）ls -al >>文件		（功能描述：列表的内容追加到文件aa.txt的末尾）
（3）cat 文件1 > 文件2	（功能描述：将文件1的内容覆盖到文件2）
（4）echo “内容” >> 文件
- 软链接也称为符号链接，类似于windows里的快捷方式，有自己的数据块，主要存放了链接其他文件的路径。
1）基本语法：
ln -s [原文件或目录] [软链接名]		（功能描述：给原文件创建一个软链接）
2）经验技巧
删除软链接： rm -rf 软链接名，而不是rm -rf 软链接名/
查询：通过ll就可以查看，列表属性第1位是l，尾部会有位置指向
- history 查看已经执行过历史命令
- date 显示当前时间
1.基本语法：
	 - date								（功能描述：显示当前时间）
	 - date +%Y							（功能描述：显示当前年份）
     - date +%m							（功能描述：显示当前月份）
     - date +%d							（功能描述：显示当前是哪一天）
	 - date "+%Y-%m-%d %H:%M:%S"		（功能描述：显示年月日时分秒）
     - date -d '1 days ago'			（功能描述：显示前一天时间）
     - date -d '-1 days ago'			（功能描述：显示明天时间）

- date 设置系统时间
1. 基本语法：
	date -s 字符串时间

2. 设置系统当前时间
```shell
[root@aliyun ~]# date -s "2017-06-19 20:52:18"
```
- cal 查看日历
- useradd 添加新用户
useradd 用户名			（功能描述：添加新用户）
useradd -g 组名 用户名	（功能描述：添加新用户到某个组）
- passwd 设置用户密码
passwd 用户名	（功能描述：设置用户密码）
- id 查看用户是否存在
```shell
[root@aliyun ~]# id root
uid=0(root) gid=0(root) groups=0(root)

```
- cat  /etc/passwd 查看创建了哪些用户
```shell
[tools@uatgw01 ~]$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin

```
- su username 切换用户
- userdel 删除用户
userdel -r 删除用户和用户目录
- whoami 查看当前登录用户名

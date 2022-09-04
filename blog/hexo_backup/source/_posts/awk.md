---
title: linux awk 教程
date: 2021-01-13 10:54:48
tags:
---


## 用以下信息做示例
```shell
$ cat netstat.txt
Proto Recv-Q Send-Q Local-Address          Foreign-Address             State
tcp        0      0 0.0.0.0:3306           0.0.0.0:*                   LISTEN
tcp        0      0 0.0.0.0:80             0.0.0.0:*                   LISTEN
tcp        0      0 127.0.0.1:9000         0.0.0.0:*                   LISTEN
tcp        0      0 coolshell.cn:80        124.205.5.146:18245         TIME_WAIT
tcp        0      0 coolshell.cn:80        61.140.101.185:37538        FIN_WAIT2
tcp        0      0 coolshell.cn:80        110.194.134.189:1032        ESTABLISHED
tcp        0      0 coolshell.cn:80        123.169.124.111:49809       ESTABLISHED
tcp        0      0 coolshell.cn:80        116.234.127.77:11502        FIN_WAIT2
tcp        0      0 coolshell.cn:80        123.169.124.111:49829       ESTABLISHED
tcp        0      0 coolshell.cn:80        183.60.215.36:36970         TIME_WAIT
tcp        0   4166 coolshell.cn:80        61.148.242.38:30901         ESTABLISHED
tcp        0      1 coolshell.cn:80        124.152.181.209:26825       FIN_WAIT1
tcp        0      0 coolshell.cn:80        110.194.134.189:4796        ESTABLISHED
tcp        0      0 coolshell.cn:80        183.60.212.163:51082        TIME_WAIT
tcp        0      1 coolshell.cn:80        208.115.113.92:50601        LAST_ACK
tcp        0      0 coolshell.cn:80        123.169.124.111:49840       ESTABLISHED
tcp        0      0 coolshell.cn:80        117.136.20.85:50025         FIN_WAIT2
tcp        0      0 :::22                  :::*                        LISTEN
```
### 输出第一列和第四列
```shell
[root@aliyun ~]# awk '{print $1,$4}' netstat.txt 
$ 
Proto Local-Address
tcp 0.0.0.0:3306
tcp 0.0.0.0:80
tcp 127.0.0.1:9000
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp coolshell.cn:80
tcp :::22
```
其中单引号中的被大括号括着的就是awk的语句，注意，其只能被单引号包含。
其中的$1..$n表示第几例。注：$0表示整个行。
### 格式化输出，类似于 c语言的printf
```shell
$ awk '{printf "%-8s %-8s %-8s %-18s %-22s %-15s\n",$1,$2,$3,$4,$5,$6}' netstat.txt
Proto    Recv-Q   Send-Q   Local-Address      Foreign-Address        State
tcp      0        0        0.0.0.0:3306       0.0.0.0:*              LISTEN
tcp      0        0        0.0.0.0:80         0.0.0.0:*              LISTEN
tcp      0        0        127.0.0.1:9000     0.0.0.0:*              LISTEN
tcp      0        0        coolshell.cn:80    124.205.5.146:18245    TIME_WAIT
tcp      0        0        coolshell.cn:80    61.140.101.185:37538   FIN_WAIT2
tcp      0        0        coolshell.cn:80    110.194.134.189:1032   ESTABLISHED
tcp      0        0        coolshell.cn:80    123.169.124.111:49809  ESTABLISHED
tcp      0        0        coolshell.cn:80    116.234.127.77:11502   FIN_WAIT2
tcp      0        0        coolshell.cn:80    123.169.124.111:49829  ESTABLISHED
tcp      0        0        coolshell.cn:80    183.60.215.36:36970    TIME_WAIT
tcp      0        4166     coolshell.cn:80    61.148.242.38:30901    ESTABLISHED
tcp      0        1        coolshell.cn:80    124.152.181.209:26825  FIN_WAIT1
tcp      0        0        coolshell.cn:80    110.194.134.189:4796   ESTABLISHED
tcp      0        0        coolshell.cn:80    183.60.212.163:51082   TIME_WAIT
tcp      0        1        coolshell.cn:80    208.115.113.92:50601   LAST_ACK
tcp      0        0        coolshell.cn:80    123.169.124.111:49840  ESTABLISHED
tcp      0        0        coolshell.cn:80    117.136.20.85:50025    FIN_WAIT2
tcp      0        0        :::22              :::*                   LISTEN
```

<!-- more -->

### 过滤
#### 过滤第三列的值为0 && 第六列的值为 LISTEN
```shell
$ awk '$3==0 && $6=="LISTEN" ' netstat.txt
tcp        0      0 0.0.0.0:3306               0.0.0.0:*              LISTEN
tcp        0      0 0.0.0.0:80                 0.0.0.0:*              LISTEN
tcp        0      0 127.0.0.1:9000             0.0.0.0:*              LISTEN
tcp        0      0 :::22                      :::*                   LISTEN
```
== 为比较运算符，其他比较运算符: !=,>,<,>=,<=
```shell
$ awk ' $3>0 {print $0}' netstat.txt
Proto Recv-Q Send-Q Local-Address          Foreign-Address             State
tcp        0   4166 coolshell.cn:80        61.148.242.38:30901         ESTABLISHED
tcp        0      1 coolshell.cn:80        124.152.181.209:26825       FIN_WAIT1
tcp        0      1 coolshell.cn:80        208.115.113.92:50601        LAST_ACK
```
### 需要表头的话，可以引入内建变量NR:
$ awk '$3==0 && $6=="LISTEN" || NR==1 ' netstat.txt
Proto Recv-Q Send-Q Local-Address          Foreign-Address             State
tcp        0      0 0.0.0.0:3306           0.0.0.0:*                   LISTEN
tcp        0      0 0.0.0.0:80             0.0.0.0:*                   LISTEN
tcp        0      0 127.0.0.1:9000         0.0.0.0:*                   LISTEN
tcp        0      0 :::22                  :::*                        LISTEN

#### 加上格式化输出
```shell
$ awk '$3==0 && $6=="LISTEN" || NR==1 {printf "%-20s %-20s %s\n",$4,$5,$6}' netstat.txt
Local-Address        Foreign-Address      State
0.0.0.0:3306         0.0.0.0:*            LISTEN
0.0.0.0:80           0.0.0.0:*            LISTEN
127.0.0.1:9000       0.0.0.0:*            LISTEN
:::22                :::*                 LISTEN
```
awk 内建变量
|  变量    |  含义    |
| ---- | ---- |
|  $0  | 当前记录(这个变量存放着整个行的内容)     |
|  $1-$n    | 当前记录的第n个字段，字段间由 FS分隔     |
|  FS    | 输入字段分隔符，默认是空格或者tab     |
| NF     |  当前记录中的字段个数，就是有多少列    |
| NR     |   已经读出的记录数，就是行号，从1开始，如果有多个文件的话，这个值也是不断累加中   |
|  FNR    | 当前记录数，与NR不同的是，这个值是各个文件自己的行号     |
|  RS    | 输入的记录分隔符，默认为换行符     |
|  OFS    | 输出字段分隔符，默认也是空格     |
|  ORS    | 输出的记录分隔符，默认为换行符     |
|  FILENAME | 当前输入文件的名字             |

### 使用行号
```shell
$ awk '$3==0 && $6=="ESTABLISHED" || NR==1 {printf "%02s %s %-20s %-20s %s\n",NR, FNR, $4,$5,$6}' netstat.txt
01 1 Local-Address        Foreign-Address      State
07 7 coolshell.cn:80      110.194.134.189:1032 ESTABLISHED
08 8 coolshell.cn:80      123.169.124.111:49809 ESTABLISHED
10 10 coolshell.cn:80      123.169.124.111:49829 ESTABLISHED
14 14 coolshell.cn:80      110.194.134.189:4796 ESTABLISHED
17 17 coolshell.cn:80      123.169.124.111:49840 ESTABLISHED
```
### 指定分隔符
```shell
$  awk  'BEGIN{FS=":"} {print $1,$3,$6}' /etc/passwd
root 0 /root
bin 1 /bin
daemon 2 /sbin
adm 3 /var/adm
lp 4 /var/spool/lpd
sync 5 /sbin
shutdown 6 /sbin
halt 7 /sbin
```
上面的命令也等价于：（-F的意思就是指定分隔符）

`$ awk  -F: '{print $1,$3,$6}' /etc/passwd`
注：如果你要指定多个分隔符，你可以这样来：
`awk -F '[;:]'`
### 使用\t 作为分隔符 (下面使用了/etc/passwd 文件，这个文件是以: 分隔的)

```shell
[root@aliyun ~]# awk -F: '{print $1,$3,$6} OFS="\t" ' /etc/passwd
root 0 /root
root:x:0:0:root:/root:/bin/bash
bin	1	/bin
bin:x:1:1:bin:/bin:/sbin/nologin
daemon	2	/sbin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm	3	/var/adm
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp	4	/var/spool/lpd
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync	5	/sbin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown	6	/sbin
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt	7	/sbin
halt:x:7:0:halt:/sbin:/sbin/halt
mail	8	/var/spool/mail
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator	11	/root
operator:x:11:0:operator:/root:/sbin/nologin
games	12	/usr/games
games:x:12:100:games:/usr/games:/sbin/nologin
ftp	14	/var/ftp
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody	99	/
nobody:x:99:99:Nobody:/:/sbin/nologin
systemd-network	192	/
systemd-network:x:192:192:systemd Network Management:/:/sbin/nologin
dbus	81	/
dbus:x:81:81:System message bus:/:/sbin/nologin
polkitd	999	/
polkitd:x:999:998:User for polkitd:/:/sbin/nologin
sshd	74	/var/empty/sshd
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
postfix	89	/var/spool/postfix
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
chrony	998	/var/lib/chrony
chrony:x:998:996::/var/lib/chrony:/sbin/nologin
nscd	28	/
nscd:x:28:28:NSCD Daemon:/:/sbin/nologin
tcpdump	72	/
tcpdump:x:72:72::/:/sbin/nologin
rpc	32	/var/lib/rpcbind
rpc:x:32:32:Rpcbind Daemon:/var/lib/rpcbind:/sbin/nologin
rpcuser	29	/var/lib/nfs
rpcuser:x:29:29:RPC Service User:/var/lib/nfs:/sbin/nologin
nfsnobody	65534	/var/lib/nfs
nfsnobody:x:65534:65534:Anonymous NFS User:/var/lib/nfs:/sbin/nologin
avahi	70	/var/run/avahi-daemon
avahi:x:70:70:Avahi mDNS/DNS-SD Stack:/var/run/avahi-daemon:/sbin/nologin
tss	59	/dev/null
tss:x:59:59:Account used by the trousers package to sandbox the tcsd daemon:/dev/null:/sbin/nologin
nginx	997	/var/lib/nginx
nginx:x:997:994:Nginx web server:/var/lib/nginx:/sbin/nologin
fsg	1000	/home/fsg
fsg:x:1000:1000::/home/fsg:/bin/bash
hadoop	1001	/home/hadoop
hadoop:x:1001:1001::/home/hadoop:/bin/bash
dockerroot	996	/var/lib/docker
dockerroot:x:996:993:Docker User:/var/lib/docker:/sbin/nologin
dolphinscheduler	1002	/home/dolphinscheduler
dolphinscheduler:x:1002:1002::/home/dolphinscheduler:/bin/bash
```
### 字符串匹配
```shell
$ awk '$6 ~ /FIN/ || NR==1 {print NR,$4,$5,$6}' OFS="\t" netstat.txt
1       Local-Address   Foreign-Address State
6       coolshell.cn:80 61.140.101.185:37538    FIN_WAIT2
9       coolshell.cn:80 116.234.127.77:11502    FIN_WAIT2
13      coolshell.cn:80 124.152.181.209:26825   FIN_WAIT1
18      coolshell.cn:80 117.136.20.85:50025     FIN_WAIT2

$ $ awk '$6 ~ /WAIT/ || NR==1 {print NR,$4,$5,$6}' OFS="\t" netstat.txt
1       Local-Address   Foreign-Address State
5       coolshell.cn:80 124.205.5.146:18245     TIME_WAIT
6       coolshell.cn:80 61.140.101.185:37538    FIN_WAIT2
9       coolshell.cn:80 116.234.127.77:11502    FIN_WAIT2
11      coolshell.cn:80 183.60.215.36:36970     TIME_WAIT
13      coolshell.cn:80 124.152.181.209:26825   FIN_WAIT1
15      coolshell.cn:80 183.60.212.163:51082    TIME_WAIT
18      coolshell.cn:80 117.136.20.85:50025     FIN_WAIT2
```
上面的第一个示例匹配FIN状态， 第二个示例匹配WAIT字样的状态。其实 ~ 表示模式开始。/ /中是模式。这就是一个正则表达式的匹配。
### awk 可以像 grep 一样去匹配第一行
```shell
[root@aliyun ~]# awk '/ESTABLISHED/' netstat.txt 
tcp        0      0 coolshell.cn:80        110.194.134.189:1032        ESTABLISHED
tcp        0      0 coolshell.cn:80        123.169.124.111:49809       ESTABLISHED
tcp        0      0 coolshell.cn:80        123.169.124.111:49829       ESTABLISHED
tcp        0   4166 coolshell.cn:80        61.148.242.38:30901         ESTABLISHED
tcp        0      0 coolshell.cn:80        110.194.134.189:4796        ESTABLISHED
tcp        0      0 coolshell.cn:80        123.169.124.111:49840       ESTABLISHED

```
### 可以用 "/FIN|TIME/" 来匹配 FIN 或者 TIME
```shell
[root@aliyun ~]# awk '$6 ~ /FIN|TIME/ || NR==1 {print NR,$4,$5,$6}' OFS="\t"   netstat.txt
1			
6	coolshell.cn:80	124.205.5.146:18245	TIME_WAIT
7	coolshell.cn:80	61.140.101.185:37538	FIN_WAIT2
10	coolshell.cn:80	116.234.127.77:11502	FIN_WAIT2
12	coolshell.cn:80	183.60.215.36:36970	TIME_WAIT
14	coolshell.cn:80	124.152.181.209:26825	FIN_WAIT1
16	coolshell.cn:80	183.60.212.163:51082	TIME_WAIT
19	coolshell.cn:80	117.136.20.85:50025	FIN_WAIT2

```
### 模式取反
```shell
[root@aliyun ~]#  awk '$6 !~ /WAIT/ || NR==1 {print NR,$4,$5,$6}' OFS="\t" netstat.txt
1			
2	Local-Address	Foreign-Address	State
3	0.0.0.0:3306	0.0.0.0:*	LISTEN
4	0.0.0.0:80	0.0.0.0:*	LISTEN
5	127.0.0.1:9000	0.0.0.0:*	LISTEN
8	coolshell.cn:80	110.194.134.189:1032	ESTABLISHED
9	coolshell.cn:80	123.169.124.111:49809	ESTABLISHED
11	coolshell.cn:80	123.169.124.111:49829	ESTABLISHED
13	coolshell.cn:80	61.148.242.38:30901	ESTABLISHED
15	coolshell.cn:80	110.194.134.189:4796	ESTABLISHED
17	coolshell.cn:80	208.115.113.92:50601	LAST_ACK
18	coolshell.cn:80	123.169.124.111:49840	ESTABLISHED
20	:::22	:::*	LISTEN


```
或是：
`awk '!/WAIT/' netstat.txt`
### 拆分文件
awk 通过重定向拆分文件，下面按第6例分隔文件(其中的NR!=1表示不处理表头)

```shell
[root@aliyun awk]# awk 'NR!=1{print > $6}' netstat.txt 
[root@aliyun awk]# ls
ESTABLISHED  FIN_WAIT1  FIN_WAIT2  LAST_ACK  LISTEN  netstat.txt  State  TIME_WAIT
[root@aliyun awk]# cat ESTABLISHED 
tcp        0      0 coolshell.cn:80        110.194.134.189:1032        ESTABLISHED
tcp        0      0 coolshell.cn:80        123.169.124.111:49809       ESTABLISHED
tcp        0      0 coolshell.cn:80        123.169.124.111:49829       ESTABLISHED
tcp        0   4166 coolshell.cn:80        61.148.242.38:30901         ESTABLISHED
tcp        0      0 coolshell.cn:80        110.194.134.189:4796        ESTABLISHED
tcp        0      0 coolshell.cn:80        123.169.124.111:49840       ESTABLISHED

```
### 把指定的列输出到文件
awk 'NR!=1{print $4,$5 > $6}' netstat.txt
再复杂一点：（注意其中的if-else-if语句，可见awk其实是个脚本解释器）
awk 'NR!=1{if($6 ~ /TIME|ESTABLISHED/) print > "1.txt";
else if($6 ~ /LISTEN/) print > "2.txt";
else print > "3.txt" }' netstat.txt

### 统计
计算所有的 c 文件，cpp文件和 h 文件的文件大小总和
 ls -l  *.cpp *.c *.h | awk '{sum+=$5} END {print sum}'

统计各个 connection 状态
 awk 'NR!=1{a[$6]++;} END {for (i in a) print i ", " a[i];}' netstat.txt

 统计各个用户占用的内存
```shell
 [root@aliyun awk]#  ps aux | awk 'NR!=1{a[$1]+=$6;} END { for(i in a) print i ", " a[i]"KB";}'rpc, 1140KB
chrony, 1556KB
polkitd, 604816KB
dbus, 1920KB
avahi, 2064KB
root, 981756KB
```
#### awk 脚本
在上面我们可以看到一个END关键字。END的意思是“处理完所有的行的标识”，即然说到了END就有必要介绍一下BEGIN，这两个关键字意味着执行前和执行后的意思，语法如下：

BEGIN{ 这里面放的是执行前的语句 }
END {这里面放的是处理完所有的行后要执行的语句 }
{这里面放的是处理每一行时要执行的语句}

假设有这么一个文件（学生成绩表）:
```shell
[root@aliyun awk]# cat score.txt 
Marry   2143 78 84 77
Jack    2321 66 78 45
Tom     2122 48 77 71
Mike    2537 87 97 95
Bob     2415 40 57 62

```
awk脚本如下:
```shell
$ cat cal.awk
#!/bin/awk -f
#运行前
BEGIN {
    math = 0
    english = 0
    computer = 0

    printf "NAME    NO.   MATH  ENGLISH  COMPUTER   TOTAL\n"
    printf "---------------------------------------------\n"
}
#运行中
{
    math+=$3
    english+=$4
    computer+=$5
    printf "%-6s %-6s %4d %8d %8d %8d\n", $1, $2, $3,$4,$5, $3+$4+$5
}
#运行后
END {
    printf "---------------------------------------------\n"
    printf "  TOTAL:%10d %8d %8d \n", math, english, computer
    printf "AVERAGE:%10.2f %8.2f %8.2f\n", math/NR, english/NR, computer/NR
}
```
看下执行结果
```shell
[root@aliyun awk]# awk -f cal.awk score.txt 
NAME    NO.   MATH  ENGLISH  COMPUTER   TOTAL
---------------------------------------------
Marry  2143     78       84       77      239
Jack   2321     66       78       45      189
Tom    2122     48       77       71      196
Mike   2537     87       97       95      279
Bob    2415     40       57       62      159
---------------------------------------------
  TOTAL:       319      393      350 
AVERAGE:     63.80    78.60    70.00

```
### 环境变量
和环境变量交互(使用-v参数和ENVIRON，使用ENVIRON的环境变量需要 export)
```shell
[root@aliyun awk]# x=5
[root@aliyun awk]# y=10
[root@aliyun awk]# export y
[root@aliyun awk]# echo $x $y
5 10
[root@aliyun awk]# awk -v val=$x '{print $1, $2, $3, $4+val, $5+ENVIRON["y"]}' OFS="\t" score.txt
Marry	2143	78	89	87
Jack	2321	66	83	55
Tom	2122	48	82	81
Mike	2537	87	102	105
Bob	2415	40	62	72

```
### practice
```shell
#从file文件中找出长度大于80的行
awk 'length>80' file

#按连接数查看客户端IP
netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr

#打印99乘法表
seq 9 | sed 'H;g' | awk -v RS='' '{for(i=1;i<=NF;i++)printf("%dx%d=%d%s", i, NR, i*NR, i==NR?"\n":"\t")}' 
```

内建变量，参看：http://www.gnu.org/software/gawk/manual/gawk.html#Built_002din-Variables
流控方面，参看：http://www.gnu.org/software/gawk/manual/gawk.html#Statements
内建函数，参看：http://www.gnu.org/software/gawk/manual/gawk.html#Built_002din
正则表达式，参看：http://www.gnu.org/software/gawk/manual/gawk.html#Regexp
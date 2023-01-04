# linux-基础命令-实战技巧

linux 命令大全：[https://www.runoob.com/linux/linux-command-manual.html](https://www.runoob.com/linux/linux-command-manual.html "https://www.runoob.com/linux/linux-command-manual.html")

# 1.xargs常用

xargs -l1 -P5 -i&#x20;

行，线程，-i命名

# 2.wait等待

等待主脚本内的所有进程跑完，才继续跑

```sql
function await() {
  for pid in $(jobs -p)
  do
  wait $pid || exit 1
  done
}
```

# 3.查看目录

查看目录大小之du（使用了多少）

du -sh 目录占用大小

df -h  总容量已用

# 4.shell 得到两个文件的差集(交集和并集)

最牛 sort a b b | uniq -u  求交集

**交集**

comm命令默认输出为三列，第一列为是A-B（在A中但不在B中），第二列B-A，第三列为A交B。-1代表不显示第一列，依次类推。

comm -12 A.txt B.txt 或者 cat a b | sort | uniq -d > c # c is a intersect b 交集

如果A B 无序，可以使用下面命令

comm <(sort a.txt|uniq ) <(sort b.txt|uniq ) -23

# 5.添加某个用户到某个组

sudo gpasswd -a \$USER work #将登陆用户加入到work用户组中&#x20;

newgrp work #更新用户组

# 6.tar.gz,.gz解压命令

tar.gz文件解压

tar　　-zxvf 　　java.tar.gz

或者解压到指定的目录里

tar　　-zxvf　　java.tar.gz　　-C　　./java

gz文件的解压 gzip 命令

gzip　　-d　　java.gz

查看命令

zcat　　java.gz

# 7.cpu占用排序

ps aux --sort=-pcpu | head -10

# 8.内存占用排序

ps aux --sort -rss | head -10

# 9.killall杀死关键字的进程

ps -ef | grep x.hql | grep -v grep | cut -c 9-15  | xargs kill -9  &#x20;

killall  + name，等同于  ps -ef | grep name | grep -v grep | cut -c 9-15  | xargs kill -9  &#x20;

即杀死同名的进程

# 10.shell脚本执行报错：/bin/bash^M: bad interpreter: No such file or directory

是由于win和linux的文件格式版本不一，解决方式如下：

1.sed -i "s/\r//" filename 或sed -i "s/^M//" filename，直接将回车符替换为空字符串。

2.vim filename，编辑文件，执行“: set ff=unix”，将文件设置为unix格式，然后执行“:wq”，保存退出。

3.dos2unix filename或busybox dos2unix filename，如果提示command not found，可以使用前两种方法。

# 11.获取当前脚本执行路径

```bash
basepath=$(cd `dirname $0`; pwd)
```

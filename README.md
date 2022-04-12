## 基于场景的文件权限管理
我们提出了一种基于场景的文件权限管理模型设计，该模型允许智能汽车用户根据应用程序在不同场景下对用户文件的使用情况配置安全策略，给应用程序分配受上下文限制的权限，而非直接赋予静态的访问权限，这将在现有研究基础上进一步将权限最小化。在我们的设计中，一旦当前上下文信息与用户定义策略中的预定义上下文匹配，就会自动授予相应的应用程序以访问权限。

## 安装
在编写和测试Demo时涉及的各软件如下所示：
	
| Setting	Related Information |Related Information  |
|--|--|
|Hypervisor|VMware Workstation 16|
| Operating System | Ubuntu 20.04LTS |
|Kernel|Linux 5.13.0-30（主操作系统内核）|
|Others|openssh-server、libncurses5-dev、bison、flex、vim、libssl-dev、libelf-dev、git、fakeroot、ncurses-dev、xz-utils、libssl-de|

在使用Demo时，需要下载[linux-4.19.163源码](https://www.kernel.org)，并拷贝到虚拟机的/usr/src目录下。然后将其security文件夹替换为我们所提供的security文件夹，最后编译这个修改后的linux-4.19.163内核源码。内核编译相关的注意事项请见附件1。

## 使用
虚拟机开机时，选择Linux 4.19.163，等待开机完成
![在这里插入图片描述](https://img-blog.csdnimg.cn/f5e87eff24b944c9936aab6682181a63.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5LiN5LyadmVjdG9y,size_20,color_FFFFFF,t_70,g_se,x_16)
开机完成后，打开终端，输入以下指令，观察到打印出形如“app is security./usr/bin/grep”的信息，说明Demo已安装成功：
```bash
dmesg | grep security | tail -10
```
接下来挂载伪文件系统，用于管理存储在内核空间中的上下文信息。首先在文件/etc/fstab末尾添加：

```bash
demofs /demo demofs defaults 0 0
```
这样每次开机，系统会自动挂载demofs伪文件系统。但是这一次开机还是要手动挂载：
```bash
sudo mount /demo
```
通过ls和cat命令可以看到相关的一些信息，表示挂载成功。
![在这里插入图片描述](https://img-blog.csdnimg.cn/0cd4aa0c88b74fb68cd9c0cf7ea1e409.png)
该Demo的功能是对文件进行基于场景的访问控制。首先我们选择一个文件用做访问控制客体。
将该文件设置为受保护文件的命令(filename需改为选取文件的名)为：

```bash
sudo setfattr -n security.protected -v 1 filename
```
授予app访问该文件的权限，假设限制条件为Cond
```bash
sudo setfattr -n security.appname -v Cond filename
```
查看上下文信息的命令为：

```bash
cat /demo/context
```
修改内核中的上下文信息需要在超级用户下执行（假设需要修改其为0，0，1，0）：

```bash
echo "imminent_collision:0;incomming_phone_call:0;high_speed:1;navigation_selected:0" > /demo/context
```

系统中默认设置了4个上下文信息（每个的值为0或1，上下文信息的数量可以通过CONTEXT_NUM修改），故Cond应当形如(0,0,0,0)，Cond中第i个数字表示对每第i个上下文信息的要求为0，1，或没有要求（2）。(Cond中的数字取值有三种：0,1,2)

撤销和修改权限只需
```bash
撤销所有权限：
sudo setfattr -n security.appname -v " " filename

添加受场景限制的权限（限制条件为Cond2），不同条件之间用“，”分隔，Cond与Cond2为“或”的关系,满足其一即可获得访问权限
sudo setfattr -n security.appname -v "Cond,Cond2" filename
```

测试功能的一个例子如下所示：

```bash
xin@xin-virtual-machine:~/桌面/code_test$ touch secret
xin@xin-virtual-machine:~/桌面/code_test$ echo hello,world > secret
xin@xin-virtual-machine:~/桌面/code_test$ cat secret
hello,world
xin@xin-virtual-machine:~/桌面/code_test$ sudo setfattr -n security.protected -v 1 secret
xin@xin-virtual-machine:~/桌面/code_test$ cat secret
cat: secret: 权限不够

xin@xin-virtual-machine:~/桌面/code_test$ sudo setfattr -n security./usr/bin/cat -v "(0,0,0,0)" secret
xin@xin-virtual-machine:~/桌面/code_test$ getfattr -n security./usr/bin/cat secret
# file: secret
security./usr/bin/cat="(0,0,0,0)"
xin@xin-virtual-machine:~/桌面/code_test$ cat /demo/context
imminent_collision:0
incomming_phone_call:0
high_speed:0
navigation_selected:0
xin@xin-virtual-machine:~/桌面/code_test$ cat secret
hello,world

xin@xin-virtual-machine:~/桌面/code_test$ sudo su
root@xin-virtual-machine:/home/xin/桌面/code_test# echo "imminent_collision:0;incomming_phone_call:0;high_speed:1;navigation_selected:0" > /demo/context
root@xin-virtual-machine:/home/xin/桌面/code_test# exit
exit
xin@xin-virtual-machine:~/桌面/code_test$ cat secret
cat: secret: 权限不够
```


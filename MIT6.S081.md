课程官网：https://pdos.csail.mit.edu/6.828/2020/xv6.html
课程翻译：https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081
xv6文档：https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf

**本文档用于记录学习过程中的关键点**

# Lec01 Introduction and examples
## 1.2 操作系统结构
![Alt text](./image/MIT6.S081/image.jpg)
* 一个操作系统只有一个kernel，用于管理每一个用户空间进程
* 系统调用与程序中的函数调用看起来是一样的，但区别是系统调用会实际运行到系统内核中，并执行内核中对于系统调用的实现。

## 1.5 read, write, exit系统调用

* **文件描述符**：0，1，2对应标准输入、标准输出和标准错误流

* read接收3个参数：

1. 第一个参数是文件描述符，指向一个之前打开的文件。Shell会确保默认情况下，当一个程序启动时，文件描述符0连接到console的输入，文件描述符1连接到了console的输出。所以我可以通过这个程序看到console打印我的输入。当然，这里的程序会预期文件描述符已经被Shell打开并设置好。

1. read的第二个参数是指向某段内存的指针，程序可以通过指针对应的地址读取内存中的数据，这里的指针就是代码中的buf参数。在代码第10行，程序在栈里面申请了64字节的内存，并将指针保存在buf中，这样read可以将数据保存在这64字节中。

1. read的第三个参数是代码想读取的最大长度，sizeof(buf)表示，最多读取64字节的数据，所以这里的read最多只能从连接到文件描述符0的设备，也就是console中，读取64字节的数据。

## 1.6 open系统调用
```c
int fd = open("output.txt", 0_WRONLY|0_CREATE);
write(fd, "ooo/n", 4);
```
* open系统调用会返回一个新分配的文件描述符，这里的文件描述符是一个小的数字，可能是2，3，4或者其他的数字。
* write的第二个参数是数据的指针，第三个参数是要写入的字节数
* **文件描述符**本质上对应了内核中的一个表单数据。内核维护了每个运行进程的状态，内核会为每一个运行进程保存一个表单，表单的key是文件描述符。这个表单让内核知道，每个文件描述符对应的实际内容是什么。这里比较关键的点是，每个进程都有自己独立的文件描述符空间，所以如果运行了两个不同的程序，对应两个不同的进程，如果它们都打开一个文件，它们或许可以得到相同数字的文件描述符，但是因为内核为每个进程都维护了一个独立的文件描述符空间，这里相同数字的文件描述符可能会对应到不同的文件。

## 1.7 Shell
Q:有一个系统调用和编译器的问题。编译器如何处理系统调用？生成的汇编语言是不是会调用一些由操作系统定义的代码段？
A:有一个特殊的RISC-V指令，程序可以调用这个指令，并将控制权交给内核。所以，实际上当你运行C语言并执行例如open或者write的系统调用时，**从技术上来说，open是一个C函数，但是这个函数内的指令实际上是机器指令**，也就是说我们调用的open函数并不是一个C语言函数，它是**由汇编语言实现**，组成这个系统调用的汇编语言实际上在RISC-V中被称为ecall。这个特殊的指令将控制权转给内核。之后内核检查进程的内存和寄存器，并确定相应的参数。

## 1.8 fork系统调用 
```c {.line-numbers} 
//////////////////////////////////
//  fork.c: create a new process

#include "kernel/types.h"
#include "user/user.h"

int
main()
{
    int pid;

    pid = fork();  // 子父进程各返回一个pid

    printf("fork() returned %d\n", pid);  // 子父进程同时执行 

    if (pid == 0){ // pid=0, 为子进程
        printf("child\n");
    } else {  // 父进程
        printf("parent\n");
    }

    exit(0);
}
```
* fork会拷贝当前进程的内存，并创建一个新的进程，这里的内存包含了进程的指令和数据。**同时也会拷贝文件描述符的表单！**
* fork在原始的进程中会返回大于0的整数，这个是新创建进程的ID。而在新创建的进程中，fork系统调用会返回0。

##  1.9 exec, wait系统调用
```c
char *argv[] = { "echo", "this", "is", "redirected", "echo", 0 };  // 0作为一个NULL指针告诉内核数组结束了
exec("echo", argv);
printf("exec failed!\n");
exit(0);
```
* ```exec("echo", argv);```操作系统从名为echo的文件中加载指令到当前的进程中，并替换了当前进程的内存，之后开始执行这些新加载的指令。同时，可以传入命令行参数,在这里就是设置好的一个字符指针的数组argv，这里的字符指针本质就是一个字符串（string）。
* exec系统调用会**保留当前的文件描述符表单**。所以任何在exec系统调用之前的文件描述符，例如0，1，2等。它们在新的程序中表示相同的东西。
*  通常来说exec系统调用**不会返回**，因为exec会完全替换当前进程的内存，相当于当前进程不复存在了，所以exec系统调用已经没有地方能返回了。它只会在kernel不能运行相应的文件时返回。
```c {.line-numbers}
//////////////////////////////
#include "kernel/types.h"
#include "user/user.h"

// forkexec.c: fork then exec

int
main()
{
    int pid. status;

    pid = fork();
    if(pid == 0){
        char *argv[] = { "echo", "this", "is", "redirected", "echo", 0 }; 
        exec("echo", argv);   // echo执行完后退出，之后父进程重新获得对子进程的控制（相当于自动执行exit(0);)
        printf("exec failed!\n");   // 如果exec未能成功执行，执行第16,17行
        exit(1);
    } else {
        printf("parent waiting\n");
        wait(&status);  // 操作系统会将1从退出的子进程传递到此处
        printf("the chiled exited with status %d\n", status);
    }

    exit(0);
}
```
输出：
![Alt text](./image/MIT6.S081/forkexec.png)
* wait会等待之前创建的子进程退出，wait的参数status，是一种让退出的子进程以一个整数（32bit的数据）的格式与等待的父进程通信方式。
* 对于Unix系统，如果一个程序成功的退出了，那么exit的参数会是0
* ```&status```，是将status对应的地址传递给内核，内核会向这个地址写入子进程向exit传入的参数(1或0)。
* Q: 当我们说子进程从父进程拷贝了所有的内存，这里具体指的是什么呢？是不是说子进程需要重新定义变量之类的？
A： 在编译之后，你的C程序就是一些在内存中的指令，这些指令存在于内存中。所以这些指令可以被拷贝，因为它们就是内存中的字节，它们可以被拷贝到别处。通过一些有关虚拟内存的技巧，可以使得子进程的内存与父进程的内存一样，这里实际就是将父进程的内存镜像拷贝给子进程，并在子进程中执行。
* Q：如果父进程有多个子进程，wait是不是会在第一个子进程完成时就退出？这样的话，还有一些与父进程交错运行的子进程，是不是需要有多个wait来确保所有的子进程都完成？
A: 是的，如果一个进程调用fork两次，如果它想要等两个子进程都退出，它需要调用wait两次。每个wait会在一个子进程退出时立即返回。当wait返回时，你实际上没有必要知道哪个子进程退出了，但是wait返回了子进程的进程号，所以在wait返回之后，你就可以知道是哪个子进程退出了。

## 1.10 I/O Redirect
```c {.line-numbers}
// redirect.c: run a command with output redirected

int
main()
{
	int pid;

	pid = fork();
	if (pid == 0) {
		close(1);  // 子进程关闭文件描述符为1的标准输出流。
		open("output.txt", 0_WRONLY|0_CREATE);   // O_WRONLY 表示写入方式，O_CREATE 表示如果文件不存在则创建

		char *argv[] = { "echo", "this", "is", "redirected", "echo", 0 };
		exec("echo", argv);
		printf("exec failed!\n");
		exit(1);
	} else {
		wait((int *) 0);
	}

	exit(0);
}
```
Shell:
```c
$ redirect
$  // no output
$ cat output.txt
this is redirected echo
```
* 代码第11行的open一定会返回1，因为**open会返回当前进程未使用的最小文件描述符序号**。因为我们刚刚关闭了文件描述符1，而文件描述符0还对应着console的输入，所以open一定可以返回1。在代码第16行之后，文件描述符1与文件output.txt关联。
* 执行```exec(echo)```时，echo会输出到文件描述符1，也就是文件output.txt。这里有意思的地方是，echo根本不知道发生了什么，**echo**也没有必要知道I/O重定向了，它**只是将自己的输出写到了文件描述符1**。只有Shell知道I/O重定向了

## 补充：pipe, list
```c {.line-numbers}
// pipe1.c: communication over a pipe

#include "kernel/types.h"
#include "user/user.h"

int
main()
{
  int fds[2];
  char buf[100];
  int n;

  // create a pipe, with two FDs in fds[0], fds[1].
  pipe(fds);

  write(fds[1], "this is pipe1\n", 14);   // 向管道的写端写入14个字节的数据,即字符串"this is pipe1\n"
  n = read(fds[0], buf, sizeof(buf));  // 从管道的读端读取数据，最多读取sizeof(buf)（即100）个字节，并将数据存储在缓冲区buf中。函数返回实际读取到的字节数，并将其存储在变量n中。

  write(1, buf, n);  // 将读取到的数据写到标准输出（文件描述符1），输出的字节数为n

  exit(0);
}
```
* 管道本质上是一个在内存中开辟的缓冲区，它连接了两个文件描述符：**第一个用于读取，第二个用于写入**。
* ```int pipe(int pipefd[2]);``` 参数：```pipefd[0]```将成为读端的文件描述符，```pipefd[1]```将成为写端的文件描述符（操作系统会自动找到当前可用的最小的两个未使用的文件描述符）
*  ```pipe()```成功时返回值为0，失败时返回值为-1，并设置errno以指示错误类型

```c {.line-numbers}
#include "kernel/types.h"
#include "user/user.h"

// pipe2.c: communication between two processes

int
main()
{
  int n, pid;
  int fds[2];
  char buf[100];

  // create a pipe, with two FDs in fds[0], fds[1].
  pipe(fds);

  pid = fork();
  if (pid == 0) {
    write(fds[1], "this is pipe2\n", 14);
  } else {
    n = read(fds[0], buf, sizeof(buf));
    write(1, buf, n);
  }

  exit(0);
}
```
* 上面代码演示了pipe在两个进程之间进行通信
```c {.line-numbers}
#include "kernel/types.h"
#include "user/user.h"

// list.c: list file names in the current directory

struct dirent {
  ushort inum;
  char name[14];
};  // 定义了一个结构体 dirent，用于表示目录项（directory entry）。每个目录项包含一个 inum 字段表示索引节点号（inode number），以及一个长度为 14 的字符数组 name 表示文件名。

int
main()
{
  int fd;  // 用于存储打开目录的文件描述符
  struct dirent e;  // 用于存储读取到的目录项

  fd = open(".", 0);
  while(read(fd, &e, sizeof(e)) == sizeof(e)){
    if(e.name[0] != '\0'){
      printf("%s\n", e.name);
    }
  }
  exit(0);
}

```
# 版本控制
## 1.  配置Linux系统HOSTS（针对虚拟机无法访问github,443错误）
1.1 打开hosts文件
```
sudo gedit /etc/hosts
```
1.2 将下面文本添加至hosts文件末尾
```
#github
140.82.114.4 github.com
151.101.1.6 github.global.ssl.fastly.net
151.101.65.6 github.global.ssl.fastly.net
151.101.129.6 github.global.ssl.fastly.net
151.101.193.6 github.global.ssl.fastly.net
```
1.3 保存并关闭文件，断开虚拟机网络然后重新连接即可

## 2. 配置Linux系统SSH（针对用https clone 远程仓库后每次push需要输密码/密码错误）
2.1 在Linux系统中生成ECDSA密钥的（*rsa密钥由于安全性问题已被github停用*）
```
//执行命令之后需要连续按3次回车键
//生成的公私钥github_id_ecdsa.pub与github_id_ecdsa文件位于root\.ssh目录下
ssh-keygen -t ecdsa -b 521 -C "450969657@qq.com"  -f ~/.ssh/github_id_ecdsa 
```
2.2 获取公钥内容
```
cd ~/.ssh
cat github_id_ecdsa.pub
```
复制保存其中的内容
2.3 将SSH key添加到ssh-agent **(须在项目文件夹中也运行一遍！)**
启动ssh-agent
```
$ eval "$(ssh-agent -s)"
> Agent pid 99360
```
添加SSH密钥到ssh-agent
```
ssh-add ~/.ssh/github_id_ecdsa 
> Identity added: /root/.ssh/github_id_ecdsa (/root/.ssh/github_id_ecdsa)
```
2.4 添加SSHkey到github账户
复制1.2中内容到网页端的个人账户设置里

2.5 测试
```
ssh -T git@github.com
> Hi bakulafalls! You've successfully authenticated, but GitHub does not provide shell access.
```

## 3. 拉取实验代码，推送
3.1 首先将mit的实验代码克隆到本地
```git clone git://g.csail.mit.edu/xv6-labs-2020```
3.2 在github创建一个新的空仓库
![Alt text](./image/MIT6.S081/image.png)
3.3 添加git仓库地址
查看 本地仓库git配置文件
```
cd xv6-labs-2020/
cat .git/config
> [core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = git://g.csail.mit.edu/xv6-labs-2020
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
[branch "util"]
	remote = origin
	merge = refs/heads/util
```
添加自己的github仓库地址
```
git remote add github git@github.com:bakulafalls/xv6-labs-2020.git
cat .git/config
>  [core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = git://g.csail.mit.edu/xv6-labs-2020
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
[branch "util"]
	remote = origin
	merge = refs/heads/util
[remote "github"]
	url = git@github.com:bakulafalls/xv6-labs-2020.git
	fetch = +refs/heads/*:refs/remotes/github/*
```
3.4 在项目目录下进行一遍2.3的操作

3.5 将实验代码推送github仓库
```c
// 将实验1用到的util分支推送到github
git checkout util
git push github util:util
```
3.6 建议每个实验创建一个测试分支，例如对于util来说
```c
git checkout util         // 切换到util分支
git checkout -b util_test // 建立并切换到util的测试分支
```
当在***util_test***分支中每测试通过一个作业，请提交（```git commit```）代码，并将所做的修改合并（```git merge```）到util中，然后提交（```git push```）到github
```c
git add .
git commit -m "完成了第一个作业"
git checkout util
git merge util_test
git push github util:util
```

# Lab1: Xv6 and Unix utilities
## **Task1: Launch xv6**
参考网站：[课程官方实验部署指南](https://pdos.csail.mit.edu/6.828/2018/tools.html)， [CENTOS7部署经验](https://blog.csdn.net/weixin_46803360/article/details/128116051?ops_request_misc=&request_id=&biz_id=102&utm_term=CENTOSMIT6.828&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-0-128116051.142^v100^pc_search_result_base1&spm=1018.2226.3001.4187)， [Ubuntu部署经验](https://blog.csdn.net/u013573243/article/details/129403949?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522171627969616800197084566%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D&request_id=171627969616800197084566&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-3-129403949-null-null.142^v100^pc_search_result_base1&utm_term=Error%3A%20Couldnt%20find%20a%20riscv64%20version%20of%20GCC%2Fbinutils.%20***%20To%20turn%20off%20this%20error%2C%20run%20gmake%20TOOLPREFIX%3D%20....&spm=1018.2226.3001.4187)
自用环境：CentOS7(VMware) + https://github.com/mit-pdos/xv6-public
实验用：```$ git clone git://g.csail.mit.edu/xv6-labs-2020```
**编译错误：**
1. ![Alt text](./image/MIT6.S081/bug1.png)
1. install riscv tool-gnu-toolchain 时，make时之前添加的PATH中的环境变量不在了，需重新在控制台输入
```
# source ~/.bash_profile
# echo $PATH
> /usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin:/root/bin:/root/bin:/opt/riscv/bin
```
或者在.bashrc中设置环境变量

**clone 错误：**
```
error: RPC failed; result=18, HTTP code = 20000 KiB/s   
fatal: The remote end hung up unexpectedly
fatal: 过早的文件结束符（EOF）
fatal: index-pack failed
```
解决方法：
  ```
  git config --global http.postBuffer 524288000
  ```

  **toolchain make 错误：**
 1. ``` log2’不是‘std’的成员```原因是Linux系统中C标准库版本过低
[解决方法](https://blog.csdn.net/m0_43453853/article/details/109381488?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522171634022916800184171915%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D&request_id=171634022916800184171915&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-4-109381488-null-null.142^v100^pc_search_result_base1&utm_term=make%20%E6%8A%A5%E9%94%99Log2%E4%B8%8D%E6%98%AFstd%E7%9A%84%E6%88%90%E5%91%98&spm=1018.2226.3001.4187)
1. 
```
g++: fatal error: 已杀死 signal terminated program cc1plus
compilation terminated.
```
原因：内存不足
尝试在VMware中将内存设为8G

**xv6-labs-2020 make 错误：**
1. 
```
user/sh.c: In function 'runcmd':
user/sh.c:58:1: error: infinite recursion detected []8;;https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html#index-Winfinite-recursion-Werror=infinite-recursion]8;;]
```
参照[记录MIT6.s081 编译QEMU中的错误](https://blog.csdn.net/weixin_51472360/article/details/128800041),修改user/sh.c:58处，添加__attribute__((noreturn))
```c {.line-numbers}
diff --git a/user/sh.c b/user/sh.c
index 83dd513..c96dab0 100644
--- a/user/sh.c
+++ b/user/sh.c
@@ -54,6 +54,7 @@ void panic(char*);
 struct cmd *parsecmd(char*);
 
 // Execute cmd.  Never returns.
+__attribute__((noreturn))
 void
 runcmd(struct cmd *cmd)
 {
```
2. make: qemu-system-riscv64：命令未找到
新版的实验代码是在riscv上编译的，所以对应的qemu也需要安装riscv版本
[方法参考](https://stackoverflow.com/questions/66718225/qemu-system-riscv64-is-not-found-in-package-qemu-system-misc)

![Alt text](./image/MIT6.S081/22cac1d518fb42380afc55cd34078a5.png)
启动成功

## Task2 Sleep
<span style="background-color: yellow;">实现xv6的UNIX程序```sleep```：您的```sleep```应该暂停到用户指定的计时数。一个滴答(tick)是由xv6内核定义的时间概念，即来自定时器芯片的两个中断之间的时间。您的解决方案应该在文件***user/sleep.c***中。</span>

**提示：**
* 在你开始编码之前，请阅读《book-riscv-rev1》的第一章

* 看看其他的一些程序（如 ***/user/echo.c, /user/grep.c, /user/rm.c***）查看如何获取传递给程序的命令行参数

* 如果用户忘记传递参数，```sleep```应该打印一条错误信息

* 命令行参数作为字符串传递; 您可以使用```atoi```将其转换为数字（详见 ***/user/ulib.c***）

* 使用系统调用```sleep```

* 请参阅kernel/sysproc.c以获取实现```sleep```系统调用的xv6内核代码（查找```sys_sleep```），***user/user.h***提供了```sleep```的声明以便其他程序调用，用汇编程序编写的***user/usys.S***可以帮助```sleep```从用户区跳转到内核区。

* 确保```main```函数调用```exit()```以退出程序。

* 将你的```sleep```程序添加到***Makefile***中的```UPROGS```中；完成之后，```make qemu```将编译您的程序，并且您可以从xv6的shell运行它。

* 看看Kernighan和Ritchie编著的《C程序设计语言》（第二版）来了解C语言。

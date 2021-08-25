# Daemon 守护进程

我们希望服务器能启动后就长期服务，直到关机，独立工作，不被某个控制终端掌握（不会被Ctrl+C (SIGINT信号)，Ctrl+\\ (SIGQUIT信号)干掉）。守护进程(Daemon Process)实现了服务器需求。

## 背景知识 进程组与会话

一群进程组成进程组，一群进程组组成会话。

进程组组号与进程组首进程（进程组组长）进程号相同，会话号与会话首进程（会话组长）相同。

进程组和会话信息，保存在检测控制块中，随fork复制默认不变。

### shell（控制终端）job（作业）

当我们在shell中输入一条命令，默认进行前台作业。shell为每个作业创建一个独立的进程组。进程组号就是作业的中某个父进程的进程号。

每个shell，至多有一个前台进程组，前台进程组运行时，就不能在shell中启动新的前台作业了（也没有作业提示符号）。

![image-20210819155749556](Daemon%20%E5%AE%88%E6%8A%A4%E8%BF%9B%E7%A8%8B.assets/image-20210819155749556.png)

前台进程通过输入输出和shell交互。除此之外，在键盘上输入Ctrl+C向前台进程组中的每个进程发送SIGINT信号，默认终止前台作业；输入Ctrl+\\向前台进程组中的每个进程发送SIGQUIT信号，默认终止前台作业；输入Ctrl+Z向前台进程组中的每个进程发送SIGTSTP信号，默认挂起前台作业。

每个shell，可以有多个后台作业。可以通过命令，转换前后台作业。

### process group（进程组），session（会话）操作函数接口

```c
//获得本进程pid号
pid_t getpid(void);

//获取本进程进程组号
pid_t getpgrp(void);

//获得pid号进程进程组号
pid_t getpgid(pid_t pid);

//设置pid号进程进程组号为pgid
int setpgid(pid_t pid, pid_t pgid);

//获得pid号进程会话号
pid_t getsid(pid_t pid);

//本进程自己独立成一个进程组
pid_t setsid(void);

```

具体api细节看man文档。这里只重点讲解最后一个接口。

>setsid()  creates  a  new  session  if  the  calling process is not a process group leader.  The calling process is the leader of the new session (i.e., its session ID is made the same as its process ID).  The calling  process  also becomes the process group leader of a new process group in the session (i.e., its process group ID is made the same as its process ID). 
>
>进程调用setid()将会建立了一个新会话，然后本进程既是会话组长，也是进程组长。禁止进程组长进程调用该接口。
>
>The calling process will be the only process in the new process group and in the new session. Initially, the new session has no controlling terminal.
>
>进程调用setid()后会变成新会话和新进程组的唯一成员。新会话没有对应的控制终端。

## 守护进程实现过程

下面介绍如何实现一个守护进程

### 执行一个 fork()，之后父进程退出，子进程继续执行。

父进程退出，是前台进程退出，终端才会继续出终端提示符，用户才能正常继续在终端中输入命令，终端继续工作。

### 子进程调用 setsid() 开启一个新会话。

注意，不能用父进程开启`setid()`。因为shell命令产生的进程，默认都会开启一个新的进程组，所以此时父进程是一个进程组组长。在man文档中，明确表示禁止进程组长开新的进程，并且给出了解释。

> Disallowing a process  group leader from calling setsid() prevents the possibility that a process group leader places itself in a new session while other processes in the process group remain in the original session; such a scenario would break  the strict two-level hierarchy of sessions and process groups.
>
> 禁止进程组长调用segid()生成新session是为了防止原有进程组中还有进程在原会话中。这样会破坏严格的会话-进程组-进程三层结构。

子进程调用`setid()`，自己建了一个新会话， 变成新会话和新进程组的唯一成员。然后本进程既是会话组长，也是进程组长。脱离的终端控制，爽得一笔。

### 可选步骤

本节的步骤根据需要做。

#### 清除进程的 umask 以确保当守护进程创建文件和目录时拥有所需的权限。

给自己加权限干活了。

 #### 修改进程的当前工作目录，通常会改为根目录（/）。

修改工作目录了，完全脱离原来的工作区了。不一定要去根目录，可以去需要的地方。

 #### 关闭守护进程从其父进程继承而来的文件描述符。新建自己的文件描述符表

在关闭了文件描述符0、1、2之后，守护进程通常会打开/dev/null 并使用dup2()  使所有这些描述符指向这个设备。

###  核心业务逻辑

现在就可以开始写自己的核心业务逻辑了。

## 样例程序

```c
/*
    写一个守护进程，每隔2s获取一下系统时间，将这个时间写入到磁盘文件中。
*/

#include <stdio.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/time.h>
#include <signal.h>
#include <time.h>
#include <stdlib.h>
#include <string.h>
int fd;

void work(int num) {
    // 捕捉到信号之后，获取系统时间，写入磁盘文件
    time_t tm = time(NULL);
    struct tm * loc = localtime(&tm);
    char * str = asctime(loc);
    write(fd ,str, strlen(str));
}

int main() {

    // 1.创建子进程，退出父进程
    pid_t pid = fork();

    if(pid > 0) {
        exit(0);
    }

    // 2.将子进程重新创建一个会话
    setsid();

    // 3.设置掩码
    umask(022);

    // 4.更改工作目录
    chdir("/home/zhuansunyuxiang/trainning/lesson28");

    // 5. 关闭、重定向文件描述符
    int fd1 = open("/dev/null", O_RDWR);
    dup2(fd1, STDIN_FILENO);
    dup2(fd1, STDOUT_FILENO);
    dup2(fd1, STDERR_FILENO);

    

    // 6.业务逻辑
    fd = open("time.txt", O_RDWR | O_CREAT | O_APPEND, 0664);

    // 捕捉定时信号
    struct sigaction act;
    act.sa_flags = 0;
    act.sa_handler = work;
    sigemptyset(&act.sa_mask);
    sigaction(SIGALRM, &act, NULL);

    struct itimerval val;
    val.it_value.tv_sec = 2;
    val.it_value.tv_usec = 0;
    val.it_interval.tv_sec = 2;
    val.it_interval.tv_usec = 0;

    // 创建定时器
    setitimer(ITIMER_REAL, &val, NULL);

    // 不让进程结束
    while (1){
        pause();
    }

    return 0;
}
```


# Linux Signal（信号）

一个信号就是一条小消息。用来通知进程发生了某个事。它是一种高形式的软件异常。

## 理念

一个信号就是一条小消息。用来通知进程发生了某个事。

什么时候会有信号呢？

* 显式请求：当我在shell中跑一个程序，程序正在执行，我按下了 Ctrl+C，就会给所有的前台进程发2号信号SIGINT，进程收到这个信号默认就会终止进程。我们可以在shell中发9号信号SIGKILL，强制杀死进程。

* 硬件错误：系统运行程序发现当前进程执行了除以0的操作，发来8号SIGFPE 信号，访问了非法内存，发来11号SIGSEGV信号。

* 外部信号：子进程退出，向父进程发送17号信号SIGCHLD，告知父进程退出。

。。。。。。

信号可以通知进程有些事发生了，也可能是硬件异常，也可能是其他进程主动发送的告知信号。进程可以处理这些信号。

我的Linux系统上，信号共有这些：

```
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```

（注意是没有32和33号的。34号之前的信号，是不可数信号，常用。后面是可数信号，不常用，甚至未定义，就不加介绍了。不同的系统中信号编号可能会略有不同，所以写程序时推荐用宏值指代他们。）

34号之前的信号，都是不排队的，进程只能看到，有这种事情发生了，并不能知道，这种事发生了几件。这些信号，都有默认的处理方式。

信号的 5 种默认处理方式

* Term 终止进程 
* Ign 当前进程忽略掉这个信号
* Core 终止进程，并生成一个Core文件 
* Stop 暂停当前进程 
* Cont 继续执行当前被暂停的进程

具体各信号的介绍：

![img](Linux%20Signal%EF%BC%88%E4%BF%A1%E5%8F%B7%EF%BC%89.assets/915116-20170516204058697-1677295653.png)

用户也可以自定义函数作为信号的处理方式。

综上，信号可以用来告知进程有事情发生了，等待进程处理。前34个信号都是不排队的。系统对这些信号有默认的处理方式，但用户也可以自己设置函数捕捉处理信号。

## 实现

![信号在内核中的表示示意图](Linux%20Signal%EF%BC%88%E4%BF%A1%E5%8F%B7%EF%BC%89.assets/signal.internal.png)

在进程p的内核中，由两个数提供信号实现。一个是pending（未决），一个是block（阻塞）。这两个数的每个位，分别对应一个信号。

举例：pending的第k位为1，说明有第k个信号待处理。block第i位是1，进程会装看不见i信号。

当内核把进程p从内核模式切换到用户模式时（如系统调用返回或上下文切换）。内核检查进程p未被block的信号集合（pending&~block）。如果为0，就说明没有信号需要处理，程序继续回用户态执行。如果非0，内核就选择集合中的某个信号k（通常是最小的信号），强制接受信号k，对信号进行处理。信号处理完成后，如果没有退出进程，就继续回用户态运行。

![image-20210818092419273](Linux%20Signal%EF%BC%88%E4%BF%A1%E5%8F%B7%EF%BC%89.assets/image-20210818092419273.png)

## 接口

这些接口都可以通过

```
 man xxx
```

 在Linux命令行中英文帮助文档，比我写的好多了。

### 主动发送信号 kill raise abort

```c++
#include <sys/types.h>
#include <signal.h>
int kill(pid_t pid, int sig);
    - 功能：给任何的进程或者进程组pid, 发送任何的信号 sig
    - 参数：
        - pid ：
            > 0 : 将信号发送给指定的进程
            = 0 : 将信号发送给当前的进程组
            = -1 : 将信号发送给每一个有权限接收这个信号的进程
            < -1 : 这个pid=某个进程组的ID取反 （-12345）
        - sig : 需要发送的信号的编号或者是宏值，0表示不发送任何信号
    例子，自杀：
    kill(getppid(), 9);
    kill(getpid(), 9);
    
int raise(int sig);
    - 功能：给当前进程发送信号
    - 参数：
        - sig : 要发送的信号
    - 返回值：
        - 成功 0
        - 失败 非0
    kill(getpid(), sig);   

void abort(void);
    - 功能： 发送SIGABRT信号给当前的进程，杀死当前进程，同下
    kill(getpid(), SIGABRT);
```
### 发送定时信号alarm setitimer

这两个接口可以给本进程发送14号信号SIGALRM

```c
#include <unistd.h>
unsigned int alarm(unsigned int seconds);
    - 功能：设置定时器（闹钟）。函数调用，开始倒计时，当倒计时为0的时候，
            函数会给当前的进程发送一个信号：SIGALARM
    - 参数：
        seconds: 倒计时的时长，单位：秒。如果参数为0，定时器无效（不进行倒计时，不发信号）。
                取消一个定时器，通过alarm(0)。
    - 返回值：
        - 之前没有定时器，返回0
        - 之前有定时器，返回之前的定时器剩余的时间

    - SIGALARM ：默认终止当前的进程，每一个进程都有且只有唯一的一个定时器。
        alarm(10);  -> 返回0
        过了1秒
        alarm(5);   -> 返回9

    alarm(100) -> 该函数是不阻塞的

#include <sys/time.h>
int setitimer(int which, const struct itimerval *new_value,
                    struct itimerval *old_value);

    - 功能：设置定时器（闹钟）。可以替代alarm函数。精度微妙us，可以实现周期性定时，（多久后开始，每隔多久发送一次SIGALRM）
    - 参数：
        - which : 定时器以什么时间计时
          ITIMER_REAL: 真实时间，时间到达，发送 SIGALRM   常用
          ITIMER_VIRTUAL: 用户时间，时间到达，发送 SIGVTALRM
          ITIMER_PROF: 以该进程在用户态和内核态下所消耗的时间来计算，时间到达，发送 SIGPROF

        - new_value: 设置定时器的属性
        
            struct itimerval {      // 定时器的结构体
            struct timeval it_interval;  // 每个阶段的时间，间隔时间
            struct timeval it_value;     // 延迟多长时间执行定时器
            };

            struct timeval {        // 时间的结构体
                time_t      tv_sec;     //  秒数     
                suseconds_t tv_usec;    //  微秒    
            };
       
        - old_value ：记录上一次的定时的时间参数，一般不使用，指定NULL
    
    - 返回值：
        成功 0
        失败 -1 并设置错误号
```
`setitimer`样例：

```c
struct itimerval new_value;

// 设置间隔的时间
new_value.it_interval.tv_sec = 2;
new_value.it_interval.tv_usec = 0;

// 设置延迟的时间,3秒之后开始第一次定时
new_value.it_value.tv_sec = 3;
new_value.it_value.tv_usec = 0;

int ret = setitimer(ITIMER_REAL, &new_value, NULL); 
```

### 自定义阻塞信号集合

我们可以自定义block哪些信号。

首先介绍自定义block的`sigset_t`型变量表示。其实它就是一个整型变量，不过我还是推荐定义后，使用下述函数操作变量。

以下信号集相关的函数都是对自定义的信号集进行操作。

```c
//定义
sigset_t myset;
//函数
int sigemptyset(sigset_t *set);
    - 功能：清空信号集中的数据,将信号集中的所有的标志位置为0
    - 参数：set,传出参数，需要操作的信号集
    - 返回值：成功返回0， 失败返回-1

int sigfillset(sigset_t *set);
    - 功能：将信号集中的所有的标志位置为1
    - 参数：set,传出参数，需要操作的信号集
    - 返回值：成功返回0， 失败返回-1

int sigaddset(sigset_t *set, int signum);
    - 功能：设置信号集中的某一个信号对应的标志位为1，表示阻塞这个信号
    - 参数：
        - set：传出参数，需要操作的信号集
        - signum：需要设置阻塞的那个信号
    - 返回值：成功返回0， 失败返回-1

int sigdelset(sigset_t *set, int signum);
    - 功能：设置信号集中的某一个信号对应的标志位为0，表示不阻塞这个信号
    - 参数：
        - set：传出参数，需要操作的信号集
        - signum：需要设置不阻塞的那个信号
    - 返回值：成功返回0， 失败返回-1

int sigismember(const sigset_t *set, int signum);
    - 功能：判断某个信号是否阻塞
    - 参数：
        - set：需要操作的信号集
        - signum：需要判断的那个信号
    - 返回值：
        1 ： signum被阻塞
        0 ： signum不阻塞
        -1 ： 失败
```
样例：

```c
#include <signal.h>
#include <stdio.h>

int main() {

    // 创建一个信号集
    sigset_t set;

    // 清空信号集的内容
    sigemptyset(&set);

    // 判断 SIGINT 是否在信号集 set 里
    int ret = sigismember(&set, SIGINT);
    if(ret == 0) {
        printf("SIGINT 不阻塞\n");
    } else if(ret == 1) {
        printf("SIGINT 阻塞\n");
    }

    // 添加几个信号到信号集中
    sigaddset(&set, SIGINT);
    sigaddset(&set, SIGQUIT);

    // 判断SIGINT是否在信号集中
    ret = sigismember(&set, SIGINT);
    if(ret == 0) {
        printf("SIGINT 不阻塞\n");
    } else if(ret == 1) {
        printf("SIGINT 阻塞\n");
    }

    // 判断SIGQUIT是否在信号集中
    ret = sigismember(&set, SIGQUIT);
    if(ret == 0) {
        printf("SIGQUIT 不阻塞\n");
    } else if(ret == 1) {
        printf("SIGQUIT 阻塞\n");
    }

    // 从信号集中删除一个信号
    sigdelset(&set, SIGQUIT);

    // 判断SIGQUIT是否在信号集中
    ret = sigismember(&set, SIGQUIT);
    if(ret == 0) {
        printf("SIGQUIT 不阻塞\n");
    } else if(ret == 1) {
        printf("SIGQUIT 阻塞\n");
    }

    return 0;
}
```

`sigpromask`获取与设置内核的阻塞。

```c
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
    - 功能：将自定义信号集中的数据设置到内核中（设置阻塞，解除阻塞，替换）
    - 参数：
        - how : 如何对内核阻塞信号集进行处理
            SIG_BLOCK: 将用户设置的阻塞信号集添加到内核中，内核中原来的数据不变
                假设内核中默认的阻塞信号集是mask， mask | set
            SIG_UNBLOCK: 根据用户设置的数据，对内核中的数据进行解除阻塞
                mask &= ~set
            SIG_SETMASK:覆盖内核中原来的值
        
        - set ：已经初始化好的用户自定义的信号集
        - oldset : 保存设置之前的内核中的阻塞信号集的状态，可以是 NULL
    - 返回值：
        成功：0
        失败：-1
            设置错误号：EFAULT、EINVAL

int sigpending(sigset_t *set);
    - 功能：获取内核中的未决信号集
    - 参数：set,传出参数，保存的是内核中的未决信号集中的信息。
```
样例：
```c
#include <stdio.h>
#include <signal.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    // 设置2、3号信号阻塞
    sigset_t set;
    sigemptyset(&set);
    // 将2号和3号信号添加到信号集中
    sigaddset(&set, SIGINT);
    sigaddset(&set, SIGQUIT);

    // 修改内核中的阻塞信号集
    sigprocmask(SIG_BLOCK, &set, NULL);
    
    int num = 0;
    
    while(1) {
        num++;
        // 获取当前的未决信号集的数据
        sigset_t pendingset;
        sigemptyset(&pendingset);
        sigpending(&pendingset);
    
        // 遍历前32位
        for(int i = 1; i <= 31; i++) {
            if(sigismember(&pendingset, i) == 1) {
                printf("1");
            }else if(sigismember(&pendingset, i) == 0) {
                printf("0");
            }else {
                perror("sigismember");
                exit(0);
            }
        }
    
        printf("\n");
        sleep(1);
        if(num == 10) {
            // 解除阻塞
            sigprocmask(SIG_UNBLOCK, &set, NULL);
        }
    }
    return 0;
}
```

### 设置某个信号的捕捉行为  signal sigaction

`signal`函数：

```c
#include <signal.h>
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
    - 功能：设置某个信号的捕捉行为
    - 参数：
        - signum: 要捕捉的信号
        - handler: 捕捉到信号要如何处理
            - SIG_IGN ： 忽略信号
            - SIG_DFL ： 使用信号默认的行为
            - 回调函数 :  这个函数是内核调用，程序员只负责写，捕捉到信号后如何去处理信号。
            回调函数：
                - 需要程序员实现，提前准备好的，函数的类型根据实际需求，看函数指针的定义
                - 不是程序员调用，而是当信号产生，由内核调用
                - 函数指针是实现回调的手段，函数实现之后，将函数名放到函数指针的位置就可以了。

    - 返回值：
        成功，返回上一次注册的信号处理函数的地址。第一次调用返回NULL
        失败，返回SIG_ERR，设置错误号
        
SIGKILL SIGSTOP不能被捕捉，不能被忽略。
```
`sighandler_t`是函数指针，该函数必须是一个int参数，无返回值的函数。具体使用很简单，见下例

```c
void myalarm(int num) {
    printf("捕捉到了信号的编号是：%d\n", num);
    printf("xxxxxxx\n");
}

//主程序中：
signal(SIGALRM, myalarm);
```

`sigaction`函数可以更复杂地设置自定义处理函数

```c
#include <signal.h>
int sigaction(int signum, const struct sigaction *act,
                        struct sigaction *oldact);

    - 功能：检查或者改变信号的处理。信号捕捉
    - 参数：
        - signum : 需要捕捉的信号的编号或者宏值（信号的名称）
        - act ：捕捉到信号之后的处理动作
        - oldact : 上一次对信号捕捉相关的设置，一般不使用，传递NULL
    - 返回值：
        成功 0
        失败 -1

 struct sigaction {
    // 函数指针，指向的函数就是信号捕捉到之后的处理函数
    void     (*sa_handler)(int);
    // 不常用
    void     (*sa_sigaction)(int, siginfo_t *, void *);
    // 临时阻塞信号集，在信号捕捉函数执行过程中，临时阻塞某些信号。
    sigset_t   sa_mask;
    // 使用哪一个信号处理对捕捉到的信号进行处理
    // 这个值可以是0，表示使用sa_handler,也可以是SA_SIGINFO表示使用sa_sigaction
    int        sa_flags;
    // 被废弃掉了
    void     (*sa_restorer)(void);
};
```
样例：

```c
#include <sys/time.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

void myalarm(int num) {
    printf("捕捉到了信号的编号是：%d\n", num);
    printf("xxxxxxx\n");
}

// 主程序：过3秒以后，每隔2秒钟定时一次
int main() {

    struct sigaction act;
    act.sa_flags = 0;
    act.sa_handler = myalarm;
    sigemptyset(&act.sa_mask);  // 清空临时阻塞信号集
   
    // 注册信号捕捉
    sigaction(SIGALRM, &act, NULL);

    struct itimerval new_value;

    // 设置间隔的时间
    new_value.it_interval.tv_sec = 2;
    new_value.it_interval.tv_usec = 0;

    // 设置延迟的时间,3秒之后开始第一次定时
    new_value.it_value.tv_sec = 3;
    new_value.it_value.tv_usec = 0;

    int ret = setitimer(ITIMER_REAL, &new_value, NULL); // 非阻塞的
    printf("定时器开始了...\n");

    if(ret == -1) {
        perror("setitimer");
        exit(0);
    }

    // getchar();
    while(1);

    return 0;
}
```

## 高级概念：如何正确自定义信号处理

### 正确同步

自定义的信号处理程序不知道什么时候就开始执行了，甚至可能交叉执行，可能会修改程序的全局变量，造成混乱。所以写安全的信号处理函数。下面是一些建议：

* 尽量简单
* 只调用异步调用安全的函数
* 注意对全局共享数据结构的保护

* 可以必要时在一些段落中阻塞一些信号，确保程序正常的逻辑，避免突然插入的信号处理函数造成的混乱。

我们对最后一点做个说明：

```c
void handler(int sig)
{
    int olderrno = errno;
    sigset_t mask_all, prev_all;
    pid_t pid;

    Sigfillset(&mask_all);
    while ((pid = waitpid(-1, NULL, 0)) > 0) { /* Reap child */
        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
        deletejob(pid); /* Delete the child from the job list */
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    if (errno != ECHILD)
        Sio_error("waitpid error");
    errno = olderrno;
}


int main(int argc, char **argv)
{
    int pid;
    sigset_t mask_all, prev_all;
    int n = N;  /* N = 5 */
    Sigfillset(&mask_all);
    Signal(SIGCHLD, handler);
    initjobs(); /* Initialize the job list */

    while (n--) {
        if ((pid = Fork()) == 0) { /* Child */
            Execve("/bin/date", argv, NULL);
        }
        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all); /* Parent */
        addjob(pid);  /* Add the child to the job list */
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    exit(0);
}
```

上述程序可能先执行子进程，再执行父进程，然后父进程就立即响应了SIGCHLD，先执行了delete job，等到add job的时候，其实这个job已经结束了。。。这样这个job永远不可能被delete了，出现了事实错误。

所以我们要在addjob之前，阻塞对SIGCHLD的响应。正确的main函数如下：

```c
int main(int argc, char **argv)
{
    int pid;
    sigset_t mask_all, mask_one, prev_one;
    int n = N; /* N = 5 */
    Sigfillset(&mask_all);
    Sigemptyset(&mask_one);
    Sigaddset(&mask_one, SIGCHLD);
    Signal(SIGCHLD, handler);
    initjobs(); /* Initialize the job list */

    while (n--) {
        Sigprocmask(SIG_BLOCK, &mask_one, &prev_one); /* Block SIGCHLD */
        if ((pid = Fork()) == 0) { /* Child process */
            Sigprocmask(SIG_SETMASK, &prev_one, NULL); /* Unblock SIGCHLD */
            Execve("/bin/date", argv, NULL);
        }
        Sigprocmask(SIG_BLOCK, &mask_all, NULL); /* Parent process */
	addjob(pid);  /* Add the child to the job list */
        Sigprocmask(SIG_SETMASK, &prev_one, NULL);  /* Unblock SIGCHLD */
    }
    exit(0);
}
```

### 信号不排队

pending的某个信号的标志位是1，只表明有这个信号被发来了，不清楚这个信号有几个，这在处理逻辑中要格外注意。下面做一个典型的例子：

#### 正确处理SIGCHLD

SIGCHLD来了，不知道有多少个子进程挂了，但都要全处理掉，防止僵尸进程。

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <signal.h>
#include <sys/wait.h>

void myWAIT(int num)
{
    printf("catch the signal : %d\n",num);
    int ret = waitpid(-1,NULL,WNOHANG);
    while(ret>0){
        printf("child die , pid = %d\n", ret);
        ret = waitpid(-1,NULL,WNOHANG);
    }
}

int main()
{
    pid_t pid;
    for(int i = 0;i<20;i++){
        pid = fork();
        if(pid == 0){
            break;
        }
    }
    if(pid>0){
        //catch the signal
        struct sigaction act;
        act.sa_flags = 0;
        act.sa_handler = myWAIT;
        sigemptyset(&act.sa_mask);
        sigaction(SIGCHLD, &act,NULL);
        while (1){
            printf("parent process :%d\n",getpid());
            sleep(2);
        }
    }else{
        printf("child process : %d\n",getpid());
    }
    return 0;
}
```


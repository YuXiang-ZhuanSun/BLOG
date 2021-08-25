# POSIX Threads Programming (基础多线程编程)

传统的进程，只有一个控制流，程序运行就是依次执行代码。多线程，可以在一个进程里，创造出多个控制流，同时有多份代码在运行。

## 线程理念

传统进程中，有多个任务到来，也只能依次执行。多线程，可以让我们同时在一个进程内，同时进行多个任务，在多核cpu上并行。

![单线程进程和多线程进程](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/2-1Q1021I11WX.gif)



### 线程 vs 进程

多进程也可以实现，并发（甚至并行）多个任务，为什么还创造出了多线程呢？

#### 多线程更轻量

比起创建一个进程，创建一个线程的开销更小。

建立进程，需要重建：

* 进程信息：进程控制块，文件描述符表
* 虚拟地址空间：栈，堆，共享库，全局变量，代码段

而创先新线程，只需要分配一个独立的栈和寄存器信息。其他虚拟地址空间，如堆，全局变量，代码，进程信息如进程控制块，文件描述符表都是都是共享的。

进程图：

![Process caption="TEST"](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/process.gif)

![image-20210823122859207](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/image-20210823122859207.png)

多线程图：

![Threads caption="TEST2"](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/thread.gif)

![image-20210823122926271](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/image-20210823122926271.png)

因为线程的物理开销小，所以线程的上下文切换速度远快于进程的上下文切换。

部分测试数据如下：

![image-20210823121401699](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/image-20210823121401699.png)

#### 多线程通讯更容易

同一进程的多个线程是共享堆空间和全局变量的，所以进行多线程通讯十分方便。

![threadUnsafe](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/threadUnsafe.gif)

需要注意的是，多线程运行由内核调度，代码运行顺序不受控。所以对于数据的共享过程，需要用户自行管理，保证合理有序。系统提供了各种锁机制，方便用户推进程序合理有序运行（同步）。

### 线程模式

从编程者的角度看，线程就是调用了一个独立于主程序运行的函数。

多个线程之间是平等的关系，不是树形关系。

![image-20210823123109645](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/image-20210823123109645.png)

每个线程都可以创建新线程，但创造出的线程是都是对等线程(peer thread)，彼此之间都是平等的。对等线程之间也可以相互回收。

编程者们总结了多线程的几种常见工作模式，供大家参考：

#### 经理/员工模式

主线程充当经理，负责输入输出，分配任务。新线程作为员工，完成任务。

#### 同等模式

类似于经理员工模式，但是主线程创建了新线程后，自己也去执行任务。

#### 流水线模式

一个任务被分成多阶段。不同的线程执行不同的阶段，同时执行。

## 线程管理接口

对编程者来说，线程就像新建了一个独立运行的函数。

本文主要介绍c语言中的接口(都在`pthread.h`)：通过这些接口可以创建线程，退出线程，回收线程，取消线程。

### 线程核心逻辑关系

* 开始的时候，主函数是默认的一个线程。

* 其他线程必须被程序使用`pthread_create`函数建立。

![peer_threads](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/peerThreads.gif)

* 各线程结束时，使用`pthread_exit`函数进行**线程自我退出**，等待其他线程回收或最终系统自动回收。

  （注意，任何线程如果调用了`exit`，整个进程都会直接退出所有线程都会终止。而且特别地，gcc编译时，对于`main`函数 `return 0;`，会编译成`exit(0)`，所以主函数一旦`return`，整个进程都会退出。所以建议设置`main`永久生命或退出时使用`pthread_exit`。）

* 默认情形下，各线程之间都是**joinable（可结合的）对等线程关系**，也就是可被其他线程回收，或杀死的。使用`pthread_join`函数，可以回收指定对等进程；使用`pthread_cancel`函数，可以杀死对等进程。

![joining](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/joining.gif)

* 与joinable相反，线程也可以是**detached（分离）的**。可以在新建线程时指定，或新建线程后调用`pthread_detach`函数设置。detach的线程就意味着这个线程和其他的线程说拜拜了。它不再能被其他线程回收，杀死。系统会自动在这个线程退出后释放线程资源。

### 创建新线程

![peer_threads](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/peerThreads.gif)

```c
#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine) (void *), void *arg);

    - 功能：创建一个子线程
    - 参数：
        - thread：传出参数，线程创建成功后，子线程的线程ID被写到该变量中。
        - attr : 设置线程的属性，一般使用默认值，NULL，可以使用这个参数初始化新线程是分离的。
        - start_routine : 函数指针，自定义的线程运行函数。
        - arg : 给第三个参数使用，传参
    - 返回值：
        成功：0
        失败：返回错误号。这个错误号和之前errno不太一样。
        获取错误号的信息：可以通过接口  char * strerror(int errnum);
```
线程创建本质上是独立运行了一个编程者自定义的函数`void *(*start_routine) (void *)`。这个函数会成为一个新的正在运行的线程。

这个自定义函数，需要一个无类型指针参数。可以通过强制类型转换，转化成任意类型指针，从而传递任意类型的参数。甚至可以通过传递结构体指针，传递多个参数给线程，甚至还可以通过强制类型转化，直接把指针转换成数据。

例子：

```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>
#include <unistd.h>

void * callback(void * arg) {
    printf("child thread...\n");
    printf("arg value: %d\n", *(int *)arg);
    return NULL;
}

int main() {

    pthread_t tid;
    int num = 10;

    // 创建一个子线程
    int ret = pthread_create(&tid, NULL, callback, (void *)&num);

    if(ret != 0) {
        char * errstr = strerror(ret);
        printf("error : %s\n", errstr);
    } 

    for(int i = 0; i < 5; i++) {
        printf("%d\n", i);
    }

    sleep(10);

    return 0;   // exit(0);
}
```

### 退出线程

线程在几种情况下会退出

* 调用了`pthread_exit`，显式退出线程
* 执行完自己的函数，`return` 。（注意，非`main`函数`return`是`pthread_exit`，`main`函数`return`是`exit`会终止整个程序。
* 某个线程调用了`exit`。该函数会终止整个进程。。
* 某个线程调用了`pthread_cancel`，“干掉了自己的兄弟”。

```c
void pthread_exit(void *retval);
    功能：终止一个线程，在哪个线程中调用，就表示终止哪个线程
    参数：
        retval:需要传递一个指针，作为一个返回值，其他进程通过pthread_join()回收可以获取到。不想使用就填NULL。

pthread_t pthread_self(void);
    功能：获取当前的线程的线程ID
        
int pthread_equal(pthread_t t1, pthread_t t2);
    功能：比较两个线程ID是否相等
    不同的操作系统，pthread_t类型的实现不一样，有的是无符号的长整型，有的是使用结构体去实现的,不建议使用==判断。
```
例子

```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>

void * callback(void * arg) {
    printf("child thread id : %ld\n", pthread_self());
    return NULL;    // pthread_exit(NULL);
} 

int main() {

    // 创建一个子线程
    pthread_t tid;
    int ret = pthread_create(&tid, NULL, callback, NULL);

    if(ret != 0) {
        char * errstr = strerror(ret);
        printf("error : %s\n", errstr);
    }

    // 主线程
    for(int i = 0; i < 5; i++) {
        printf("%d\n", i);
    }

    printf("tid : %ld, main thread id : %ld\n", tid ,pthread_self());

    // 让主线程退出,当主线程退出时，不会影响其他正常运行的线程。
    pthread_exit(NULL);

    printf("main thread exit\n");  //不会被执行 

    return 0;   // exit(0);
}
```

我暂时觉得某个线程调用`pthread_cancel`干掉其他线程并不是很常用。

```c
#include <pthread.h>
int pthread_cancel(pthread_t thread);
    - 功能：取消线程（让线程终止）
        取消某个线程，可以终止某个线程的运行，
        但是并不是立马终止，而是当子线程执行到一个取消点，线程才会终止。
        取消点：系统规定好的一些系统调用，我们可以粗略的理解为从用户区到内核区的切换，这个位置称之为取消点。
```
### 线程回收

新建线程默认都是joinable的，我们可以通过`pthread_join`，**阻塞地**等待某个进程完成任务返回。

![joining](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/joining.gif)

```c
#include <pthread.h>
int pthread_join(pthread_t thread, void **retval);
    - 功能：和一个已经终止的线程进行连接
            回收子线程的资源
            这个函数是阻塞函数，调用一次只能回收一个子线程
            一般在主线程中使用
    - 参数：
        - thread：需要回收的子线程的ID
        - retval: 接收子线程退出时的返回值
    - 返回值：
        0 : 成功
        非0 : 失败，返回的错误号
```
我们来研究一下`pthread_join`的`retval`。在线程退出时，`pthread_exit(void *v);`参数是一个地址。我们要通过`pthread_join`获取到它。

我们把一个地址传到`pthread_join`函数里，这个地址将来就要保存`pthread_exit(void *v)`传出来的那个地址。所以需要一个地址类型的地址参数。看下面的例子使用，还是很简单的，不需要二级指针变量。

**（c的库函数和Linux系统调用中，常使用地址类型做函数的传出参数，而不是使用变量的引用。**

```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>
#include <unistd.h>

int value = 10;

void * callback(void * arg) {
    printf("child thread id : %ld\n", pthread_self());
    // sleep(3);
    // return NULL; 
    // int value = 10; // 局部变量不能用作返回，在栈里，线程退出后地址被系统收回 
    pthread_exit((void *)&value);   // return (void *)&value;是一样的 
} 

int main() {

    // 创建一个子线程
    pthread_t tid;
    pthread_create(&tid, NULL, callback, NULL);
    // 主线程
    for(int i = 0; i < 5; i++) {
        printf("%d\n", i);
    }

    printf("tid : %ld, main thread id : %ld\n", tid ,pthread_self());

    // 主线程调用pthread_join()回收子线程的资源
    int * thread_retval;
    pthread_join(tid, (void **)&thread_retval);

    printf("exit data : %d\n", *thread_retval);

    printf("回收子线程资源成功！\n");

    return 0; //子进程都回收掉了，当然可以直接return
}
```

### 线程分离

线程要么是joinable的要么是detached。对于不需要回收参数的线程，不如让它detached，简单快乐。

有两种办法让线程是detached，第一种：初始化时使用`attr`参数设置。第二种，调用`pthread_detach`函数。

#### 设置线程属性attr

线程属性attr是用一个结构体来表示的。attr可以操作线程的很多特征，比如设置是否分离，**设置线程栈的大小**等待。

我们也不直接写这个结构体的成员，我们通过调用接口来修改这个结构体。

```c
// 创建一个线程属性变量
pthread_attr_t attr;

int pthread_attr_init(pthread_attr_t *attr);
    - 初始化线程属性变量

int pthread_attr_destroy(pthread_attr_t *attr);
    - 释放线程属性的资源

int pthread_attr_getdetachstate(const pthread_attr_t *attr, int *detachstate);
    - 获取线程分离的状态属性

int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
    - 设置线程分离的状态属性
```
例子：创建新线程时，就设置新线程是分离的。

```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>
#include <unistd.h>

void * callback(void * arg) {
    printf("chid thread id : %ld\n", pthread_self());
    return NULL;
}

int main() {

    // 创建一个线程属性变量
    pthread_attr_t attr;
    // 初始化属性变量
    pthread_attr_init(&attr);

    // 设置属性
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

    // 创建一个子线程
    pthread_t tid;
    pthread_create(&tid, &attr, callback, NULL);

    // 获取线程的栈的大小
    size_t size;
    pthread_attr_getstacksize(&attr, &size);
    printf("thread stack size : %ld\n", size);

    // 输出主线程和子线程的id
    printf("tid : %ld, main thread id : %ld\n", tid, pthread_self());

    // 释放线程属性资源
    pthread_attr_destroy(&attr);

    pthread_exit(NULL);

    return 0;
}
```

#### 线程分离函数

    #include <pthread.h>
    int pthread_detach(pthread_t thread);
        - 功能：分离一个线程。被分离的线程在终止的时候，会自动释放资源返回给系统。
          1.不能多次分离，会产生不可预料的行为。
          2.不能去连接一个已经分离的线程，会报错。
        - 参数：需要分离的线程的ID
        - 返回值：
            成功：0
            失败：返回错误号
例子：

```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>
#include <unistd.h>

void * callback(void * arg) {
    printf("chid thread id : %ld\n", pthread_self());
    return NULL;
}

int main() {

    // 创建一个子线程
    pthread_t tid;

    int ret = pthread_create(&tid, NULL, callback, NULL);

    // 输出主线程和子线程的id
    printf("tid : %ld, main thread id : %ld\n", tid, pthread_self());

    // 设置子线程分离,子线程分离后，子线程结束时对应的资源就不需要主线程释放
    ret = pthread_detach(tid);
    
    pthread_exit(NULL);

    return 0;
}
```

## 线程同步

多线程运行由内核调度，代码运行顺序不受控。所以对于数据的共享过程，需要用户自行管理，保证合理有序。系统提供了各种锁机制，方便用户推进程序合理有序运行（同步）。

举一些简单的例子证明需要锁机制。

卖飞机票。有很多人会同时查询，选购，这些人在不同的进程里，但是同一张票，卖给A了就不能卖给B。不能出现，座位1，票还在，A来买，在卖给A的过程中B又来买，系统发现座位1 还是有票状态，又把票卖给了B。---互斥锁

在公交车上，有司机和售票员，司机停车之后，告知售票员，售票员才能开车门；售票员关上车门，告诉司机，司机才能开车。---条件变量

停车场，停车位只有有限个，有几个出入口，但是绝不能多放一辆车进来。---信号量

我们一个一个来看。

### mutex 互斥量

mutex的基础思想：对于临界区的操作，只能有一个线程正在操作。每个线程进入临界区，需要获得“锁”，哪个进程抢到了锁，才能进入临界区操作，其他请求只能等待，等到锁释放之后，再次抢锁。

临界区往往是操作共享资源（堆，全局变量）的过程。

我们先看看不做互斥锁操作会怎么样。

一个简单的程序，两个线程里都是做N次cnt++;看最后的结果是不是2N。

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>
const long N = 10000000;
long cnt;

void *thread(void *vargp)
{
    long j, niters = *((long *)vargp); 
    for ( j = 0; j < niters; j++)
        cnt++;                   
    return NULL;
} 


int main()
{
    long niters;
    pthread_t tid1, tid2;

    niters = N;
    pthread_create(&tid1, NULL, thread, &niters);
    pthread_create(&tid2, NULL, thread, &niters);
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);

    /* Check result */
    if (cnt != (2 * niters))
        printf("BOOM! cnt=%ld\n", cnt);
    else
        printf("OK cnt=%ld\n", cnt);
    exit(0);
}

```

结果如下：

```shell
zhuansunyuxiang@entry-of-plct:~/trainning/atest$ ./thread
OK cnt=20000000
zhuansunyuxiang@entry-of-plct:~/trainning/atest$ ./thread
OK cnt=20000000
zhuansunyuxiang@entry-of-plct:~/trainning/atest$ ./thread
BOOM! cnt=17125324
zhuansunyuxiang@entry-of-plct:~/trainning/atest$ ./thread
BOOM! cnt=18076751
zhuansunyuxiang@entry-of-plct:~/trainning/atest$ ./thread
BOOM! cnt=19202423
```

为什么呢，cnt++;不是一个“不可分割的操作”。本质上，是cnt = cnt +1;一个线程计算完cnt+1，还没有来得及赋给cnt，另一个线程就去计算了cnt+1，然后两个线程都把cnt+1赋给了cnt，结果cnt只变成了cnt+1，其实应该变成cnt+2的。这就出现了一次计数错误。

为了避免这种悲剧，正确完成cnt计数，一个程序在做++操作的时候，另一个程序一定不能进行++操作。这样就安全了。让其他线程不能执行++部分的代码，可以用以下的互斥锁接口：

互斥量的类型：pthread_mutex_t 

```c
int pthread_mutex_init(pthread_mutex_t *restrict mutex, const pthread_mutexattr_t *restrict attr);
    - 初始化互斥量
    - 参数 ：
        - mutex ： 需要初始化的互斥量变量
        - attr ： 互斥量相关的属性，NULL
    - restrict : C语言的修饰符，被修饰的指针，不能由另外的一个指针进行操作。
        pthread_mutex_t *restrict mutex = xxx;
        pthread_mutex_t * mutex1 = mutex;

int pthread_mutex_destroy(pthread_mutex_t *mutex);
    - 释放互斥量的资源

int pthread_mutex_lock(pthread_mutex_t *mutex);
    - 加锁，阻塞的，如果有一个线程加锁了，那么其他的线程只能阻塞等待

int pthread_mutex_trylock(pthread_mutex_t *mutex);
    - 尝试加锁，如果加锁失败，不会阻塞，会直接返回。

int pthread_mutex_unlock(pthread_mutex_t *mutex);
    - 解锁，把锁让给其他进程。
```
我们来修正第一个程序。

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>
const long N = 10000000;
long cnt;
pthread_mutex_t mutex;//定义互斥量
void *thread(void *vargp)
{
    long j, niters = *((long *)vargp); 
    for ( j = 0; j < niters; j++){
        pthread_mutex_lock(&mutex);//加锁
        cnt++;
        pthread_mutex_unlock(&mutex);//解锁
    }      
    return NULL;
} 


int main()
{
    long niters;
    pthread_t tid1, tid2;

    niters = N;
    pthread_mutex_init(&mutex,NULL);//初始化互斥量
    pthread_create(&tid1, NULL, thread, &niters);
    pthread_create(&tid2, NULL, thread, &niters);
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);

    /* Check result */
    if (cnt != (2 * niters))
        printf("BOOM! cnt=%ld\n", cnt);
    else
        printf("OK cnt=%ld\n", cnt);
    exit(0);
}

```

运行结果：

```c
zhuansunyuxiang@entry-of-plct:~/trainning/atest$ gcc thread.c -o thread -pthread 
zhuansunyuxiang@entry-of-plct:~/trainning/atest$ ./thread
OK cnt=20000000
zhuansunyuxiang@entry-of-plct:~/trainning/atest$ ./thread
OK cnt=20000000
zhuansunyuxiang@entry-of-plct:~/trainning/atest$ ./thread
OK cnt=20000000
zhuansunyuxiang@entry-of-plct:~/trainning/atest$ ./thread
OK cnt=20000000
```

临界区保护成功了。

需要注意的是：本例中并没有体现出多线程的运行速度优势，因为核心语句始终只能有一个线程在运行。而且加锁和解锁的时间开销也很大。所以只能算一个语法和原理演示。在实际应用的多线程程序中，线程的大多数部分可以彼此独立执行，只有个别临界区操作是独占的，需要上锁。合理设计较小高效的临界区，是提高多线程编程效果的重要一环。

### 条件变量机制

> 只有大家关了车门，司机才能发动汽车。

只有等条件满足了，我们才能去做一些事。线程同步中就提供了这种机制：

1. 条件不足，线程阻塞，等待唤醒
2. 条件满足，唤醒线程

条件变量机制需要一个条件变量类型`pthread_cond_t`和一个互斥量类型`pthread_mutex_t`配合完成。

下面来看接口：

```c
//条件变量的类型 pthread_cond_t，互斥量类型pthread_mutex_t

//初始化一个条件变量，性质一般用默认的NULL
int pthread_cond_init(pthread_cond_t *restrict cond, const pthread_condattr_t *restrict attr);
//销毁一个条件变量，没用过
int pthread_cond_destroy(pthread_cond_t *cond);
//条件不足，进行等待
int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex);
    - 进入阻塞等待，调用了该函数，线程会阻塞，原先占有的mutex会释放出去。（注意先占有mutex才能调用这个函数
int pthread_cond_timedwait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex, const struct timespec *restrict abstime);
    - 等待多长时间，调用了这个函数，线程会阻塞，直到指定的时间结束。（没用过
//唤醒线程
int pthread_cond_signal(pthread_cond_t *cond);
    - 唤醒一个或者多个等待的线程
int pthread_cond_broadcast(pthread_cond_t *cond);
    - 唤醒所有的等待的线程
```
最后两个接口目前还不清楚区别

我们来写一个例子，经典的生产者消费者模型

#### 条件变量生产者消费者模式（缓冲区大小无限制）

生产者放进去，然后通知消费者，缓冲区中有一个新物品

消费者等待缓冲区有东西，有东西就拿走

![image-20210824124247354](POSIX%20Threads%20Programming%20(%E5%9F%BA%E7%A1%80%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B).assets/image-20210824124247354.png)

现实举例

* 多媒体过程，生产者放一帧进来，消费者放一帧进屏幕
* 事件驱动的用户界面交互：点鼠标，敲键盘
  * 这些事情会组成事件队列，系统其他部分会读取事件然后对其做出反应

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>

const int P = 5;
const int C = 5;

//定义互斥量
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;//另一种初始化方式
//定义条件变量
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;//另一种初始化方式

struct Node{
    int num;
    struct Node * next;
};

struct Node * head = NULL;

void * produce(void * arg)
{
    while (1)
    {
        struct Node * newNode = (struct Node *)malloc(sizeof(struct Node));
        newNode->num = rand() % 1000;
        usleep(100);//模拟生产过程
        printf("add node, num : %3d, tid : %ld\n", newNode->num, pthread_self());
        pthread_mutex_lock(&mutex);
        newNode->next = head;
        head = newNode;
        pthread_cond_signal(&cond);// 只要生产了一个，就通知消费者消费
        pthread_mutex_unlock(&mutex);
        //在Linux 线程中，有两个队列，分别是cond_wait队列和mutex_lock队列， cond_signal让线程从cond_wait队列移到mutex_lock队列，
        //所以我推荐先通知再解锁。如果先解锁，再通知，可能会出现在通知之前就切换到其他线程，耽误了通知。
    }
    return NULL;
}

void * consume(void * arg)
{
    while (1)
    {   
        struct Node * tmp;
        pthread_mutex_lock(&mutex);
        while (head == NULL)//必须是while，因为解锁之后必须再次检验队列是不是空的，可能有其他线程又已经把线程消耗空了。
        {
            // 没有数据，需要等待
            pthread_cond_wait(&cond,&mutex);
            // 当这个函数调用阻塞的时候，会对互斥锁进行解锁，当不阻塞的，继续向下执行，会重新加锁。
        }
        tmp = head;
        head = head->next;
        pthread_mutex_unlock(&mutex);
        printf("del node, num : %3d, tid : %ld\n", tmp->num, pthread_self());
        usleep(100);//模拟消费使用过程
    }
    
    return NULL;
}

int main()
{
    pthread_t ptid[P],ctid[C];
    for (int i = 0; i < P; i++)
    {
        pthread_create(&ptid[i],NULL,produce,NULL);
        pthread_detach(ptid[i]);
    }
    for (int i = 0; i < C; i++)
    {
        pthread_create(&ctid[i],NULL,consume,NULL);
        pthread_detach(ctid[i]);
    }
    pthread_exit(NULL);
    return 0;
}
```

* 在Linux 线程中，有两个队列，分别是cond_wait队列和mutex_lock队列， cond_signal让线程从cond_wait队列移到mutex_lock队列，所以我推荐先通知再解锁。如果先解锁，再通知，可能会出现在通知之前就切换到其他线程，耽误了通知。不过其实这个好像影响不大。
* cond_wait的条件，必须是while循环。解锁之后必须再次检验队列是不是空的，因为可能有其他线程又已经把队列消耗空了（比如cond_signal之后，唤醒了一个消费者，把队列消费空了，然后又唤醒了新的消费者，其实队列已经空了，直接爆炸。

### 信号量

> 停车场里只有有限个停车位

信号量就是允许有限个线程进入临界区。

信号量不仅仅是线程间通信，更多的是一种进程间通信的工具，但是还挺好用的，嘻嘻。

信号量的接口在`semaphore.h`

```c
信号量的类型 sem_t
int sem_init(sem_t *sem, int pshared, unsigned int value);
    - 初始化信号量
    - 参数：
        - sem : 信号量变量的地址
        - pshared : 0 用在线程间 ，非0 用在进程间
        - value : 信号量中的值

int sem_destroy(sem_t *sem);
    - 释放资源

int sem_wait(sem_t *sem);
    - 对信号量加锁，调用一次对信号量的值-1，如果值为0，就阻塞

int sem_trywait(sem_t *sem);

int sem_timedwait(sem_t *sem, const struct timespec *abs_timeout);
int sem_post(sem_t *sem);
    - 对信号量解锁，调用一次对信号量的值+1

int sem_getvalue(sem_t *sem, int *sval);
```
#### 信号量 vs 条件变量

在Posix.1基本原理一文声称，有了互斥锁和条件变量还提供信号量的原因是：“本标准提供信号量的而主要目的是提供一种进程间同步的方式；这些进程可能共享也可能不共享内存区。互斥锁和条件变量是作为线程间的同步机制说明的；这些线程总是共享(某个)内存区。这两者都是已广泛使用了多年的同步方式。每组原语都特别适合于特定的问题”。

#### 信号量生产者消费者模型（缓冲区有大小限制

缓冲区BUF_SIZE为10。`sem_t`类型变量`slot`表示缓冲区剩余空间数，`sem_t`类型变量`item`表示缓冲区中有多少个待消费物件

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>
#include <semaphore.h>
#define BUF_SIZE 10
const int P = 5;
const int C = 5;

int buffer[BUF_SIZE];
int rear = 0;
int front = 0;

pthread_mutex_t mutex;

sem_t slot;
sem_t item;

void *produce(void *arg)
{
    while (1)
    {
        int x = rand() % 1000;
        printf("add node, num : %3d, tid : %ld\n", x, pthread_self());
        sem_wait(&slot);
        pthread_mutex_lock(&mutex);
        buffer[rear++] = x;
        rear %=BUF_SIZE;
        pthread_mutex_unlock(&mutex);
        sem_post(&item);
    }
    return NULL;
}

void *consume(void *arg)
{
    while (1)
    {
        int x;
        sem_wait(&item);
        pthread_mutex_lock(&mutex);
        x = buffer[front++];
        front %=BUF_SIZE;
        pthread_mutex_unlock(&mutex);
        sem_post(&slot);
        printf("del node, num : %3d, tid : %ld\n", x, pthread_self());
    }
    return NULL;
}

int main()
{
    pthread_mutex_init(&mutex, NULL);
    sem_init(&slot, 0, BUF_SIZE);
    sem_init(&item, 0, 0);

    pthread_t ptid[P],ctid[C];
    for (int i = 0; i < P; i++)
    {
        pthread_create(&ptid[i],NULL,produce,NULL);
        pthread_detach(ptid[i]);
    }
    for (int i = 0; i < C; i++)
    {
        pthread_create(&ctid[i],NULL,consume,NULL);
        pthread_detach(ctid[i]);
    }
    pthread_exit(NULL);
    return 0;
}
```


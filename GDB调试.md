# GDB调试

GDB 是由 GNU 软件系统社区提供的调试工具，同 GCC 配套组成了一套完整的开发环境，GDB 是 Linux 和许多类 Unix 系统中的标准开发环境。 

 一般来说，GDB 主要帮助你完成下面四个方面的功能： 

1. 启动程序，可以按照自定义的要求随心所欲的运行程序 
2.  可让被调试的程序在所指定的调置的断点处停住（断点可以是条件表达式） 
3.  当程序被停住时，可以检查此时程序中所发生的事 
4.  可以改变程序，将一个 BUG 产生的影响修正从而测试其他 BUG

GDB可以用来调试c/c++程序，本文主要讲这些功能：

* 启动与关闭调试
* 单句执行
* 断点调试
* 打印变量
* 多进程调试

**加粗的文字，介绍了GDB的一些习性。**

## 启动与关闭

如果需要用gbd调试，程序在gcc编译时，需要添加`-g`参数

```shell
zhuansunyuxiang@entry-of-plct:~/trainning/atest/gdb$ gcc a.c b.c -o c -g
```

`-g` 选项的作用是在可执行文件中加入源代码的信息，比如可执行文件中第几条机器指令对应源代码的第几行，但并不是把整个源文件嵌入到可执行文件中，所以在调试时必须保证 GDB 能找到源文件。

启动GDB调试的命令：`gdb xxx`，样例：

```shell
zhuansunyuxiang@entry-of-plct:~/trainning/atest/gdb$ gdb c
```

打开GDB后命令行变成了这样：

```shell
(gdb) 
(gdb) 
(gdb) 
```

输入`q`，按回车，退出GDB调试。

**退出调试后，此次运行的设置就都全部作废了。**

## 查看代码

`list`命令，简写为`l`可以用来查看代码。

```c
//默认查看主函数
(gdb) l 
    
//可以查看某一行附近的代码
(gdb) l 99
    
//可以查看某个函数
(gdb) l b
    
//可以指定源文件名:行号或者函数名
(gdb) l b.c:b
    
//设置显示的行数
show list/listsize
set list/listsize 行数
```

演示：

```shell
(gdb) l
1       #include <stdio.h>
2       void b();
3
4       int main(){
5           b();
6           for(int i = 0;i < 10;i++){
7               printf("i = %d\n",i);
8           }
9           printf("asdf\n");
10          printf("asdf\n");
(gdb) l
11          printf("asdf\n");
12          printf("asdf\n");
13          return 0;
14      }
(gdb) l b                       //查看函数b的实现
1       #include <stdio.h>
2
3       void b(){
4           for(int j = 0;j < 10;j++){
5               printf("j = %d\n",j);
6           }
7       }
(gdb) 
```

**GDB中，很多命令在没有歧义的情况下，都可以缩写。下述如c/continue意思是continue命令可以缩写为c。**

## 让程序运行

可以给程序argv设置参数

```c
//给程序设置参数/获取设置参数
(gdb) set args 10 20
(gdb) show args
```

运行程序相关的命令：

```c
//运行GDB程序
(gdb) start（程序停在第一行）
(gdb) run（遇到断点才停）
    
//继续运行，到下一个断点停
(gdb) c/continue
    
//向下执行一行代码（不会进入函数体）
n/next
    
//变量操作
p/print 变量名（打印变量值）
ptype 变量名（打印变量类型）
    
//向下单步调试（遇到函数进入函数体）
s/step
finish（跳出函数体）
    
//自动变量操作
display 变量名（自动打印指定变量的值）
i/info display
undisplay 编号
    
//其它操作
set var 变量名=变量值 （循环中用的较多）
until （跳出循环
```

演示：**注意printf会正常输出显示。**

```c
(gdb) start
Temporary breakpoint 1 at 0x1149: file a.c, line 4.
Starting program: /home/zhuansunyuxiang/trainning/atest/gdb/c 

Temporary breakpoint 1, main () at a.c:4
4       int main(){
(gdb) step
5           b();
(gdb) step
b () at b.c:3
3       void b(){
(gdb) step
4           for(int j = 0;j < 10;j++){
(gdb) s
5               printf("j = %d\n",j);
(gdb) next
j = 0
4           for(int j = 0;j < 10;j++){
(gdb) n
5               printf("j = %d\n",j);
(gdb) n
j = 1
4           for(int j = 0;j < 10;j++){
(gdb) c
Continuing.
j = 2
j = 3
j = 4
j = 5
j = 6
j = 7
j = 8
j = 9
i = 0
i = 1
i = 2
i = 3
i = 4
i = 5
i = 6
i = 7
i = 8
i = 9
[Inferior 1 (process 4663) exited normally]
(gdb) 
```



## 断点调试

注意，在第i行设置断点，意思是在执行这一行之前暂停。

```c
//设置断点
b/break 行号
b/break 函数名
b/break 文件名:行号
b/break 文件名:函数

//查看断点
i/info b/break

//删除断点
d/del/delete 断点编号

//设置断点无效
dis/disable 断点编号

//设置断点生效
ena/enable 断点编号

//设置条件断点（一般用在循环的位置）
b/break 10 if i==5
```

示例：设置与查看断点。

```c
(gdb) list
1       #include <stdio.h>
2       void b();
3
4       int main(){
5           b();
6           for(int i = 0;i < 10;i++){
7               printf("i = %d\n",i);
8           }
9           return 0;
10      }
(gdb) break 7 if i==5        //设置一个端点在第7行，如果i==5中断
Breakpoint 1 at 0x1168: file a.c, line 7.
(gdb) info break            //查看断点信息
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000000000001168 in main at a.c:7
        stop only if i==5
(gdb) run                  //run运行直到断点
Starting program: /home/zhuansunyuxiang/trainning/atest/gdb/c 
j = 0
j = 1
j = 2
j = 3
j = 4
j = 5
j = 6
j = 7
j = 8
j = 9
i = 0
i = 1
i = 2
i = 3
i = 4

Breakpoint 1, main () at a.c:7   //i==1，在执行第7行之前暂停了
7               printf("i = %d\n",i);
(gdb) next
i = 5
6           for(int i = 0;i < 10;i++){
(gdb) next
7               printf("i = %d\n",i);
(gdb) next
i = 6
6           for(int i = 0;i < 10;i++){
(gdb) continue
Continuing.
i = 7
i = 8
i = 9
[Inferior 1 (process 6326) exited normally]
(gdb) 
```

示例：删除断点。使断点无效(disable)

```c
(gdb) l
1       #include <stdio.h>
2       void b();
3
4       int main(){
5           b();
6           for(int i = 0;i < 10;i++){
7               printf("i = %d\n",i);
8           }
9           printf("asdf\n");
10          printf("asdf\n");
(gdb) l
11          printf("asdf\n");
12          printf("asdf\n");
13          return 0;
14      }
(gdb)  break 7 if i==5
Breakpoint 1 at 0x1188: file a.c, line 7.
(gdb) break 10
Breakpoint 2 at 0x11b4: file a.c, line 10.
(gdb) break 11
Breakpoint 3 at 0x11c0: file a.c, line 11.
(gdb) info break
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000000000001188 in main at a.c:7
        stop only if i==5
2       breakpoint     keep y   0x00000000000011b4 in main at a.c:10
3       breakpoint     keep y   0x00000000000011c0 in main at a.c:11
(gdb) disable 3    //让编号为3的断点失效
(gdb) delete 2     //删除编号为2的断点
(gdb) break 12     //在第12行插入断点
Breakpoint 4 at 0x11cc: file a.c, line 12.
(gdb) info break
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000000000001188 in main at a.c:7
        stop only if i==5
3       breakpoint     keep n   0x00000000000011c0 in main at a.c:11   //第2个断点已经没了，第3个断点enable属性是no
4       breakpoint     keep y   0x00000000000011cc in main at a.c:12   //新增在12行的第4个断点
(gdb) run
Starting program: /home/zhuansunyuxiang/trainning/atest/gdb/c 
j = 0
j = 1
j = 2
j = 3
j = 4
j = 5
j = 6
j = 7
j = 8
j = 9
i = 0
i = 1
i = 2
i = 3
i = 4

Breakpoint 1, main () at a.c:7           //i == 5 断点停留了
7               printf("i = %d\n",i);
(gdb) next
i = 5
6           for(int i = 0;i < 10;i++){
(gdb) next
7               printf("i = %d\n",i);
(gdb) next
i = 6
6           for(int i = 0;i < 10;i++){
(gdb) continue 
Continuing.
i = 7
i = 8
i = 9
asdf
asdf
asdf

Breakpoint 4, main () at a.c:12           //4号断点停留了
12          printf("asdf\n");
(gdb) c
Continuing.
asdf
[Inferior 1 (process 7269) exited normally]
(gdb) 
```

**GDB对断点，执行流等自动从小到大编号，删了某一个，也不会再重复编这个号了，只是继续向更大编号。**

**我们可以对每个编号进行操作，甚至删除。**

## 多进程调试

使用 GDB 调试的时候，GDB 默认只能跟踪一个进程，可以在 fork 函数调用之前，通过指令设置 GDB 调试工具跟踪父进程或者是跟踪子进程，默认跟踪父进程。

```c
(gdb) set follow-fork-mode child //设置跟踪子进程
(gdb) set follow-fork-mode parent //设置跟踪父进程
```

GDB 默认只能跟踪一个进程，可以设置调试模式，让GDB跟踪多个进程，即关闭fork后detach（脱离控制）。

```c
(gdb) set detach-on-fork off //关闭脱离，gdb会跟踪多进程
```

关于进程调试：调整跟踪或运行哪个分支

```c
//查看调试的进程：
(gdb) info inferiors

//切换当前调试的进程：
(gdb) inferior id

//使进程脱离 GDB 调试：
(gdb) detach inferiors id
```

样例：

```c
(gdb) list
1       #include <sys/types.h>
2       #include <unistd.h>
3       #include <stdio.h>
4       void b();
5
6       int main()
7       {
8           if(fork()==0){
9               //child
10              printf("I am child %d\n",getpid());
(gdb) list
11              b();
12          }else{
13              printf("I am parent %d\n",getpid());
14              for(int i = 0;i<10;i++){
15                  printf("i=%d\n",i);
16              }
17              printf("asdf\n");
18              printf("asdf\n");
19              printf("asdf\n");
20              printf("asdf\n");
(gdb) b 11  //设了第1个断点在第11行
Breakpoint 1 at 0x11d6: file m.c, line 11.
(gdb) b 18  //设了第2个断点在第18行
Breakpoint 2 at 0x122f: file m.c, line 18.
(gdb) set detach-on-fork off   //设置默认不分离进程
(gdb) run  //开始跑了
Starting program: /home/zhuansunyuxiang/trainning/atest/gdb/m 
[New inferior 2 (process 9005)]
Reading symbols from /home/zhuansunyuxiang/trainning/atest/gdb/m...
I am parent 9001  //fork之后，先跑了父进程
i=0
i=1
i=2
i=3
i=4
i=5
i=6
i=7
i=8
i=9
asdf

Thread 1.1 "m" hit Breakpoint 2, main () at m.c:18  //停在断点2上了
18              printf("asdf\n");
(gdb) info inferiors                               //看一看有几个分支
  Num  Description       Executable        
* 1    process 9001      /home/zhuansunyuxiang/trainning/atest/gdb/m //这是父进程，*代表当前gdb在调试的进程
  2    process 9005      /home/zhuansunyuxiang/trainning/atest/gdb/m //这是子进程
(gdb) inferior 2                                  //去调试第二个分支
[Switching to inferior 2 [process 9005] (/home/zhuansunyuxiang/trainning/atest/gdb/m)]
[Switching to thread 2.1 (process 9005)]
#0  0x00007ffff7eb40bf in fork () from /lib/x86_64-linux-gnu/libc.so.6
(gdb) n
Single stepping until exit from function fork,
which has no line number information.
main () at m.c:8
8           if(fork()==0){
(gdb) n
10              printf("I am child %d\n",getpid());
(gdb) c
Continuing.
I am child 9005                              //成功运行子进程

Thread 2.1 "m" hit Breakpoint 1, main () at m.c:11
11              b();
(gdb) c
Continuing.
j = 0
j = 1
j = 2
j = 3
j = 4
j = 5
j = 6
j = 7
j = 8
j = 9
[Inferior 2 (process 9005) exited normally]   //子进程运行完退出了
(gdb) i inferiors                             //查看
  Num  Description       Executable        
  1    process 9001      /home/zhuansunyuxiang/trainning/atest/gdb/m 
* 2    <null>            /home/zhuansunyuxiang/trainning/atest/gdb/m 
(gdb) 
```


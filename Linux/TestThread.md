<!-- # 初识线程, 常用的 `pthread` 系列接口介绍 -->

<!-- 这里加个现实生活中的小例子引入一下更好. -->

现在, 什么也别问, 先看代码和现象.

代码:

```cpp
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>

void * task(void * args) {
    while (1) {
        printf("这是一个新线程\n");
        //让它打印慢一些
        sleep(1);
    }
}

int main(void) {
    pthread_t tid;
    pthread_create(&tid, NULL, &task, NULL);
    while (1) {
        printf("这是主线程\n");
        sleep(1);
    }    

    return 0;
}
```

输出:

```bash
vivit@Xen:test_thread$ ./a.out 
这是主线程
这是一个新线程
这是主线程
这是一个新线程
这是主线程
这是一个新线程
这是一个新线程
这是主线程
```

我们可以直观得感受到, `main()` 函数中的 `while` 循环和 `task()` 函数中的 `while` 循环同时被执行了. 不对啊, 我没在 `main()` 函数中调用这个函数, 它是怎么跑起来的? 

怎么样? 是不是和用 `fork()` 函数写子进程的时候特别像? 父子进程一起执行相同或不同的代码块. 甚至连执行顺序不一定这个特性都符合了. 

现在, 我们注意这行代码, 其他的先别管:

```cpp
pthread_create(&tid, NULL, &task, NULL);
```

重点看第三个参数, 它是我们的 `task()` 函数的地址. 这里 `task()` 函数唯一一处在 `main()` 函数中出现, 以上现象的原因绝对跟它脱不了干系. 

不绕圈子了, 其实这就是传说中的创建线程. 这一行代码的任务就是创建一个线程, 并让这个线程去执行 `task()` 函数. 

现在, 我们把代码升级一下:

```cpp
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#define MAX 5
void * task(void * args) {
    while (1) {
        printf("这是一个新线程: %ld\n", (long)args);
        sleep(1);
    }
}

int main(void) {
    pthread_t tid[MAX];
    for (long i = 0; i < MAX; i++) {
        pthread_create(&tid[i], NULL, &task, (void *)i);
        printf("线程%d已创建\n", tid[i]);
    }
    while (1) {
        printf("这是主线程\n");
        sleep(1);
    }    

    return 0;
}
```

这一次的输出, 明显比上一次热闹得多. 因为我们创建了5个线程. 而这5个线程都在打印.

<!-- 这个tid是什么鬼, 我到现在还没搞清楚, 好像是每个线程独立栈的起始地址来着. 回头再说吧, 先发. -->

```bash
vivit@Xen:test_thread$ ./a.out 
线程274405056已创建
这是一个新线程: 0
线程266012352已创建
这是一个新线程: 1
线程257619648已创建
这是一个新线程: 2
线程249226944已创建
这是一个新线程: 3
线程240834240已创建
这是主线程
这是一个新线程: 4
这是一个新线程: 0
```

同时我们修改了第4个参数: 

```cpp
pthread_create(&tid, NULL, &task, (void *)i);
```

在 `task()` 函数中:

```cpp
printf("这是一个新线程: %ld\n", (long)args);
```

再结合输出, 我们可以得出结论: 第4个参数, 是传给第3个函数的, 不过在传递的过程中需要适当的类型转换.

注意这里, 我们在 `for()` 循环中加了这句:

```cpp
printf("线程%d已创建\n", tid[i]);
```

进程有自己的 `pid`, 线程也有自己的 `tid`. 不过这一长串打印出来的是什么鬼? 为什么没有进程的 `pid` 那样优雅, 而是一个很大的数? 我们暂不关心, 后面再说.

通过以上内容, 我们可以推出: 创建一个线程, 就是让进程运行中, 多执行一个函数, 并且我们可以给它传参. 但指定的函数, 类型必须是 `void * (void *)`.

## 线程创建

现在我们来说一说我们刚才写的代码, 由于其他的语句全是一些打印, 循环, 没有什么好说的. 我们的重点应该放在 `pthread_create()` 函数上. 

函数原型:

```cpp
#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                  void *(*start_routine) (void *), void *arg);
```

作用: 创建线程

> 注意: 一些匪夷所思的数据类型

`pthread_t` 实际上就是一个无符号长整型, 定义如下:  

```cpp
typedef unsigned long int pthread_t;
```

`void * (*start_routine) (void *)` 这是一个函数指针 `start_routine` 它指向返回类型为空类型指针, 且参数也为空类型指针的函数.

好, 正式来看看它的参数:

- `thread`: 输出型参数, 以输出型参数的形式带回线程 `tid`
- 第二个参数, 我们暂不关心, 直接设置为 `NULL`
- `start_routine`: 给线程指定的任务(函数)的指针
- `arg`: 给任务函数的参数

通过它的原型, 我们可以看到, `pthread_create()` 函数对指定线程执行的函数是有要求的, 它必须是 `void * (void *)` 类型的, 即返回值和参数都是空类型指针.

返回值:

- 成功时: 返回0
- 失败时: 返回一个错误的值, 并且不会修改 `thread`

<!--    On success, pthread_create() returns 0; on error, it returns an error number, and the contents of
       *thread are undefined. -->

<!-- 多叫几个帮手, 工作更好做.

什么是线程? 教材上的定义大多是: **线程是进程中的执行流.** 看完之后, 让人觉得, 好像懂了, 又好像没懂.

觉得懂了是看明白了它是一个执行流, 也就是要被CPU调度和执行的. 没明白的是什么叫 "进程中的执行流"? 根据已知的知识, 进程不就是被调度执行的单位吗? 你看什么进程创建 `fork()` 系统调用, 进程 -->


<!-- ## 线程等待

## 线程分离

## 线程取消

## 线程 -->
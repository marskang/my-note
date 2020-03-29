
### 关于cpu在执行过程中为了提高效率可能交换指令的情况

&nbsp; &nbsp; 最近在看《程序员的自我修养》一书，看到线程安全的部分，发现cpu在执行过程中，为了提高效率有可能交换指令的顺序，比如下面的代码:
```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>

int x = 0, y = 0;
int a = 0, b = 0;
void* thread1(void* arg) {
    x = 1;
    a = y;
}

void* thread2(void* arg) {
    y = 1;
    b = x;
}

int main(int argc, char** argv) {
    int times = 0;
    for(; ;) {
        x = 0, y = 0, a = 0, b = 0;
        pthread_t tid1, tid2; 
        pthread_create(&tid1, NULL, (void *)thread1, NULL);
        pthread_create(&tid2, NULL, (void *)thread2, NULL);
        pthread_join(tid1, NULL);
        pthread_join(tid2, NULL);
        times++;
        if (a==0 && b==0) {
            printf("a=%d, b=%d times=%d\n", a, b, times);
            break;
        }
    }
    return 0;
}
```
从逻辑上讲，这个代码执行起来应该是一个死循环，因为单个线程里面代码都是顺序执行的，不可能出现a和b同时为0的情况。但是实际执行的过程中，却发现不是死循环，而且每次times的值都不相同，这就很诡异了。而出现a和b都为0的情况只可能是线程里面的代码交换了位置，比如:
```c
void* thread1(void* arg) {
    a = y;
    x = 1;
}
```
&nbsp;&nbsp;说明了cpu在执行过程中，为了提高效率是有可能交换指令顺序的,https://hacpai.com/article/1459654970712 该篇文章详细讲了cpu在并行执行发生乱序的原因，主要是因为cpu为了提高每个周期内执行的指令数，会将两个没有关联的指令进行交换。
&nbsp;&nbsp;该案例中一种方式是可以加锁来保障线程安全，代码如下:
```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
int x = 0, y = 0;
int a = 0, b = 0;
static pthread_mutex_t g_mutex_lock;
void* thread1(void* arg) {
    pthread_mutex_lock(&g_mutex_lock);
    x = 1;
    a = y;
    pthread_mutex_unlock(&g_mutex_lock);
}

void* thread2(void* arg) {
    pthread_mutex_lock(&g_mutex_lock);
    y = 1;
    b = x;
    pthread_mutex_unlock(&g_mutex_lock);
}

int main(int argc, char** argv) {
    int times = 0;
    pthread_mutex_init(&g_mutex_lock, NULL);
    for(; ;) {
        x = 0, y = 0, a = 0, b = 0;
        pthread_t tid1, tid2; 
        pthread_create(&tid1, NULL, (void *)thread1, NULL);
        pthread_create(&tid2, NULL, (void *)thread2, NULL);
        pthread_join(tid1, NULL);
        pthread_join(tid2, NULL);
        times++;
        if (a==0 && b==0) {
            printf("a=%d, b=%d times=%d\n", a, b, times);
            break;
        }
    }
    return 0;
}
```
&nbsp;&nbsp;另外一种方式是阻止cpu换序，pthread库中提供了屏障(barrier)这种多线程并行工作的同步机制。其实pthread_join函数就是一种屏障，允许一个线程等待，直到另一个线程退出。
&nbsp;&nbsp;pthread库中提供了pthread_barrier_init,pthread_barrier_wait,pthread_barrier_destroy这三个屏障使用的函数,调用pthread_barrier_init初始化时，需要传入count，代表在允许所有线程继续允许之前，必须到达的线程数目,调用pthread_barrier_wait的线程在屏障计数未满足条件时，会进入休眠状态。如果该线程时最后一个调用pthread_barrier_wait的线程，就满足了屏障计数，所有的线程都被唤醒，具体代码如下:
```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>

int x = 0, y = 0;
int a = 0, b = 0;
pthread_barrier_t barrier;
void* thread1(void* arg) {
    x = 1;
    pthread_barrier_wait(&barrier);
    a = y;
}

void* thread2(void* arg) {
    y = 1;
    pthread_barrier_wait(&barrier);
    b = x;
}

int main(int argc, char** argv) {
    int times = 0;
    int rc = pthread_barrier_init(&barrier, NULL, 2);
    if (rc) {
        fprintf(stderr, "pthread_barrier_init: %s\n", strerror(rc));
        exit(1);
    }
    for(; ;) {
        x = 0, y = 0, a = 0, b = 0;
        pthread_t tid1, tid2;
        pthread_create(&tid1, NULL, (void *)thread1, NULL);
        pthread_create(&tid2, NULL, (void *)thread2, NULL);
        pthread_join(tid1, NULL);
        pthread_join(tid2, NULL);
        times++;
        if (a==0 && b==0) {
            printf("a=%d, b=%d times=%d\n", a, b, times);
            break;
        }
    }
    // pthread_barrier_destroy(&barrier);

    return 0;
}
```
&nbsp;&nbsp;之前有"神牛"指出由于乱序的影响，单例模式下的DCL(double checked locking)也是靠不住的，可以看下这个文章 http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html ，其实java使用volatile即可，因为volatile就实现了jvm的内存屏障。
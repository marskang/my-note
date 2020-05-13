
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
&nbsp;&nbsp;另外一种方式是阻止cpu指令重排，阻止cpu指令重排的一种方式是使用内存屏障，X86平台上面提供了lfence，sfence，mfence指令来进行内存屏障:  
　　1. lfence，是一种Load Barrier 读屏障。在读指令前插入读屏障，可以让高速缓存中的数据失效，重新从主内存加载数据  
　　2. sfence, 是一种Store Barrier 写屏障。在写指令之后插入写屏障，能让写入缓存的最新数据写回到主内存  
　　3. mfence, 是一种全能型的屏障，具备ifence和sfence的能力  
我们这里直接使用mfence来解决指令重排的问题,代码如下:
```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#define barrier() __asm__ __volatile__("mfence": : :"memory")
int x = 0, y = 0;
int a = 0, b = 0;
void* thread1(void* arg) {
    x = 1;
    barrier();
    a = y;
}

void* thread2(void* arg) {
    y = 1;
    barrier();
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

&nbsp;&nbsp;之前有"神牛"指出由于乱序的影响，单例模式下的DCL(double checked locking)也是靠不住的，可以看下这个文章 http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html 。其实java使用volatile即可，java的volatile在c/c++的volatile(阻止编译器优化)的基础上加了lock前缀的指令来解决指令重排的问题(没有使用fence指令是为了可移植性？大部分cpu lock指令都支持)。
Lock前缀指令的能力：  
　　1. 它先对总线/缓存加锁，然后执行后面的指令，最后释放锁后会把高速缓存中的脏数据全部刷新回主内存。  
　　2. 在Lock锁住总线的时候，其他CPU的读写请求都会被阻塞，直到锁释放。Lock后的写操作会让其他CPU相关的cache line失效，从而从新从内存加载最新的数据。  
下面是hotspot关于java volatile的部分代码  
```c++
inline void OrderAccess::fence() {
  if (os::is_MP()) {
    // always use locked addl since mfence is sometimes expensive
#ifdef AMD64
    __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
#else
    __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
#endif
  }
}
```

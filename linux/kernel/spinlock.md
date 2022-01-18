# 自旋锁

## 代码路径
> include/linux/spinlock.h  
> kernel/locking/spinlock.c  


内核当发生访问资源冲突的时候，可以有两种锁的解决方案选择：

* 一个是原地等待
* 一个是挂起当前进程，调度其他进程执行（睡眠）
Spinlock 是内核中提供的一种比较常见的锁机制，自旋锁是“原地等待”的方式解决资源冲突的，即，一个线程获取了一个自旋锁后，另外一个线程期望获取该自旋锁，获取不到，只能够原地“打转”（忙等待）。由于自旋锁的这个忙等待的特性，注定了它使用场景上的限制 —— 自旋锁不应该被长时间的持有（消耗 CPU 资源）。


## 自旋锁的使用
在linux kernel的实现中，经常会遇到这样的场景：共享数据被中断上下文和进程上下文访问，该如何保护呢？如果只有进程上下文的访问，那么可以考虑使用semaphore或者mutex的锁机制，但是现在中断上下文也参和进来，那些可以导致睡眠的lock就不能使用了，这时候，可以考虑使用spin lock。

这里为什么把中断上下文标红加粗呢？因为在中断上下文，是不允许睡眠的（原因详见文章《Linux 中断之中断处理浅析》中的第四章），所以，这里需要的是一个不会导致睡眠的锁——spinlock。

换言之，中断上下文要用锁，首选 spinlock。

## 使用自旋锁，有两种方式定义一个锁
动态的：
```c
spinlock_t lock;
spin_lock_init (&lock);
```
静态的：
```c
DEFINE_SPINLOCK(lock);
```

## 自旋锁的死锁和解决
自旋锁不可递归，自己等待自己已经获取的锁，会导致死锁。

自旋锁可以在中断上下文中使用，但是试想一个场景：一个线程获取了一个锁，但是被中断处理程序打断，中断处理程序也获取了这个锁（但是之前已经被锁住了，无法获取到，只能自旋），中断无法退出，导致线程中后面释放锁的代码无法被执行，导致死锁。（如果确认中断中不会访问和线程中同一个锁，其实无所谓）

1. 考虑下面的场景（内核抢占场景）：
（1）进程A在某个系统调用过程中访问了共享资源 R
（2）进程B在某个系统调用过程中也访问了共享资源 R

会不会造成冲突呢？假设在A访问共享资源R的过程中发生了中断，中断唤醒了沉睡中的，优先级更高的B，在中断返回现场的时候，发生进程切换，B启动执行，并通过系统调用访问了R，如果没有锁保护，则会出现两个thread进入临界区，导致程序执行不正确。OK，我们加上spin lock看看如何：A在进入临界区之前获取了spin lock，同样的，在A访问共享资源R的过程中发生了中断，中断唤醒了沉睡中的，优先级更高的B，B在访问临界区之前仍然会试图获取spin lock，这时候由于A进程持有spin lock而导致B进程进入了永久的spin……怎么破？linux的kernel很简单，在A进程获取spin lock的时候，禁止本CPU上的抢占（上面的永久spin的场合仅仅在本CPU的进程抢占本CPU的当前进程这样的场景中发生）。如果A和B运行在不同的CPU上，那么情况会简单一些：A进程虽然持有spin lock而导致B进程进入spin状态，不过由于运行在不同的CPU上，A进程会持续执行并会很快释放spin lock，解除B进程的spin状态


2. 再考虑下面的场景（中断上下文场景）：
（1）运行在CPU0上的进程A在某个系统调用过程中访问了共享资源 R
（2）运行在CPU1上的进程B在某个系统调用过程中也访问了共享资源 R
（3）外设P的中断handler中也会访问共享资源 R

在这样的场景下，使用spin lock可以保护访问共享资源R的临界区吗？我们假设CPU0上的进程A持有spin lock进入临界区，这时候，外设P发生了中断事件，并且调度到了CPU1上执行，看起来没有什么问题，执行在CPU1上的handler会稍微等待一会CPU0上的进程A，等它立刻临界区就会释放spin lock的，但是，如果外设P的中断事件被调度到了CPU0上执行会怎么样？CPU0上的进程A在持有spin lock的状态下被中断上下文抢占，而抢占它的CPU0上的handler在进入临界区之前仍然会试图获取spin lock，悲剧发生了，CPU0上的P外设的中断handler永远的进入spin状态，这时候，CPU1上的进程B也不可避免在试图持有spin lock的时候失败而导致进入spin状态。为了解决这样的问题，linux kernel采用了这样的办法：如果涉及到中断上下文的访问，spin lock需要和禁止本 CPU 上的中断联合使用。

3. 再考虑下面的场景（底半部场景）
linux kernel中提供了丰富的bottom half的机制，虽然同属中断上下文，不过还是稍有不同。我们可以把上面的场景简单修改一下：外设P不是中断handler中访问共享资源R，而是在的bottom half中访问。使用spin lock+禁止本地中断当然是可以达到保护共享资源的效果，但是使用牛刀来杀鸡似乎有点小题大做，这时候disable bottom half就OK了

4. 中断上下文之间的竞争

同一种中断handler之间在uni core和multi core上都不会并行执行，这是linux kernel的特性。
如果不同中断handler需要使用spin lock保护共享资源，对于新的内核（不区分fast handler和slow handler），所有handler都是关闭中断的，因此使用spin lock不需要关闭中断的配合。
bottom half又分成softirq和tasklet，同一种softirq会在不同的CPU上并发执行，因此如果某个驱动中的softirq的handler中会访问某个全局变量，对该全局变量是需要使用spin lock保护的，不用配合disable CPU中断或者bottom half。

tasklet更简单，因为同一种tasklet不会多个CPU上并发。

## 自旋锁的实现
1. 文件整理

和体系结构无关的代码如下：
（1） include/linux/spinlock_types.h
这个头文件定义了通用spin lock的基本的数据结构（例如spinlock_t）和如何初始化的接口（DEFINE_SPINLOCK）。这里的“通用”是指不论SMP还是UP都通用的那些定义。
（2）include/linux/spinlock_types_up.h
这个头文件不应该直接include，在include/linux/spinlock_types.h文件会根据系统的配置（是否SMP）include相关的头文件，如果UP则会include该头文件。这个头文定义UP系统中和spin lock的基本的数据结构和如何初始化的接口。当然，对于non-debug版本而言，大部分struct都是empty的。
（3）include/linux/spinlock.h 这个头文件定义了通用spin lock的接口函数声明，例如spin_lock、spin_unlock等，使用spin lock模块接口API的驱动模块或者其他内核模块都需要include这个头文件。
（4）include/linux/spinlock_up.h 这个头文件不应该直接include，在include/linux/spinlock.h文件会根据系统的配置（是否SMP）include相关的头文件。这个头文件是debug版本的spin lock需要的。
（5）include/linux/spinlock_api_up.h 同上，只不过这个头文件是non-debug版本的spin lock需要的
（6）linux/spinlock_api_smp.h SMP上的spin lock模块的接口声明
（7）kernel/locking/spinlock.c SMP上的spin lock实现。

对UP和SMP上spin lock头文件进行整理：

UP需要的头文件	SMP需要的头文件
* linux/spinlock_type_up.h:
* linux/spinlock_types.h: 
* linux/spinlock_up.h: 
* linux/spinlock_api_up.h: 
* linux/spinlock.h
* asm/spinlock_types.h 
* linux/spinlock_types.h: 
* asm/spinlock.h 
* linux/spinlock_api_smp.h: 
* linux/spinlock.h

2. 数据结构

首先定义一个 spinlock_t 的数据类型，其本质上是一个整数值（对该数值的操作需要保证原子性），该数值表示spin lock是否可用。初始化的时候被设定为1。当thread想要持有锁的时候调用spin_lock函数，该函数将spin lock那个整数值减去1，然后进行判断，如果等于0，表示可以获取spin lock，如果是负数，则说明其他thread的持有该锁，本thread需要spin。

内核中的spinlock_t的数据类型定义如下：
```c
typedef struct spinlock { 
        struct raw_spinlock rlock;  
} spinlock_t;
 
typedef struct raw_spinlock { 
    arch_spinlock_t raw_lock; 
} raw_spinlock_t;
```
通用（适用于各种arch）的spin lock使用spinlock_t这样的type name，各种arch定义自己的struct raw_spinlock。听起来不错的主意和命名方式，直到linux realtime tree（PREEMPT_RT）提出对spinlock的挑战。real time linux是一个试图将linux kernel增加硬实时性能的一个分支（你知道的，linux kernel mainline只是支持soft realtime），多年来，很多来自realtime branch的特性被merge到了mainline上，例如：高精度timer、中断线程化等等。realtime tree希望可以对现存的spinlock进行分类：一种是在realtime kernel中可以睡眠的spinlock，另外一种就是在任何情况下都不可以睡眠的spinlock。分类很清楚但是如何起名字？起名字绝对是个技术活，起得好了事半功倍，可维护性好，什么文档啊、注释啊都素那浮云，阅读代码就是享受，如沐春风。起得不好，注定被后人唾弃，或者拖出来吊打（这让我想起给我儿子起名字的那段不堪回首的岁月……）。最终，spin lock的命名规范定义如下：

（1）spinlock，在rt linux（配置了PREEMPT_RT）的时候可能会被抢占（实际底层可能是使用支持PI（优先级翻转）的mutext）。
（2）raw_spinlock，即便是配置了PREEMPT_RT也要顽强的spin
（3）arch_spinlock，spin lock是和architecture相关的，arch_spinlock是architecture相关的实现

对于UP平台，所有的arch_spinlock_t都是一样的，定义如下：
```c
typedef struct { } arch_spinlock_t;
```
什么都没有，一切都是空啊。当然，这也符合前面的分析，对于UP，即便是打开的preempt选项，所谓的spin lock也不过就是disable preempt而已，不需定义什么spin lock的变量。
对于SMP平台，这和arch相关，我们在下面描述。
在具体的实现面，我们不可能把每一个接口函数的代码都呈现出来，我们选择最基础的spin_lock为例子，其他的读者可以自己阅读代码来理解。
spin_lock的代码如下：
```c
static inline void spin_lock(spinlock_t *lock) 
{ 
    raw_spin_lock(&lock->rlock); 
}
```
当然，在linux mainline代码中，spin_lock和raw_spin_lock是一样的，在这里重点看看raw_spin_lock，代码如下：
```c
#define raw_spin_lock(lock)    _raw_spin_lock(lock)
```
UP中的实现：
```c
#define _raw_spin_lock(lock)            __LOCK(lock)
 
#define __LOCK(lock) \ 
  do { preempt_disable(); ___LOCK(lock); } while (0)
```
SMP的实现：
```c
void __lockfunc _raw_spin_lock(raw_spinlock_t *lock) 
{ 
    __raw_spin_lock(lock); 
}
 
static inline void __raw_spin_lock(raw_spinlock_t *lock) 
{ 
    preempt_disable(); 
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_); 
    LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock); 
}
```
UP中很简单，本质上就是一个preempt_disable而已，SMP中稍显复杂，preempt_disable当然也是必须的，spin_acquire可以略过，这是和运行时检查锁的有效性有关的，如果没有定义CONFIG_LOCKDEP其实就是空函数。如果没有定义CONFIG_LOCK_STAT（和锁的统计信息相关），LOCK_CONTENDED就是调用 do_raw_spin_lock 而已，如果没有定义CONFIG_DEBUG_SPINLOCK，它的代码如下：
```c
static inline void do_raw_spin_lock(raw_spinlock_t *lock) __acquires(lock) 
{ 
    __acquire(lock); 
    arch_spin_lock(&lock->raw_lock); 
}
```
__acquire和静态代码检查相关，忽略之，最终实际的获取spin lock还是要靠arch相关的代码实现。


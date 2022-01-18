# 原子操作
## 代码路径
***
> include/linux/atomic.h  
> arch/arm64/include/asm/atomic.h  
> include/asm-generic/atomic-instrumented.h
***

## 原子操作简介
所谓原子操作，就是该操作绝不会在执行完毕前被任何其他任务或事件打断，也就说，它的最小的执行单位，不可能有比它更小的执行单位，因此这里的原子实际是使用了物理学里的物质微粒的概念。
原子操作需要硬件的支持，因此是架构相关的，其API和原子类型的定义都定义在内核源码树的include/asm/atomic.h文件中，它们都使用汇编语言实现，因为C语言并不能实现这样的操作。
原子操作主要用于实现资源计数，很多引用计数(refcnt)就是通过原子操作实现的。

## 原子操作api介绍

原子类型定义如下:
```c
typedef struct {
	int counter;
} atomic_t;
```

初始化并定义一个原子变量：
```c
atomic_t v = ATOMIC_INIT(0);
```
## 基本调用
Linux 为原子操作提供了基本的操作宏函数：
```c
atomic_inc(v); // 原子变量自增1
atomic_dec(v); // 原子变量自减1
atomic_read(v) // 读取一个原子量
atomic_add(int i, atomic_t *v) // 原子量增加 i
atomic_sub(int i, atomic_t *v) // 原子量减少 i
```
这里定义的都是通用入口，真正的操作和处理器架构体系相关，这里分析 ARM 架构体系的实现：
```c
static inline void atomic_inc(atomic_t *v)
{
    atomic_add_return(1, v);
}
```
## 主要api
```c
atomic_read(atomic_t * v);
```
该函数对原子类型的变量进行原子读操作，它返回原子类型的变量v的值。
```c
atomic_set(atomic_t * v, int i);
```
该函数设置原子类型的变量v的值为i。
```c
void atomic_add(int i, atomic_t *v);
```
该函数给原子类型的变量v增加值i。
```c
atomic_sub(int i, atomic_t *v);
```
该函数从原子类型的变量v中减去i。
```c
int atomic_sub_and_test(int i, atomic_t *v);
```
该函数从原子类型的变量v中减去i，并判断结果是否为0，如果为0，返回真，否则返回假。
```c
void atomic_inc(atomic_t *v);
```
该函数对原子类型变量v原子地增加1。
```c
void atomic_dec(atomic_t *v);
```
该函数对原子类型的变量v原子地减1。
```c
int atomic_dec_and_test(atomic_t *v);
```
该函数对原子类型的变量v原子地减1，并判断结果是否为0，如果为0，返回真，否则返回假。
```c
int atomic_inc_and_test(atomic_t *v);
```
该函数对原子类型的变量v原子地增加1，并判断结果是否为0，如果为0，返回真，否则返回假。
```c
int atomic_add_negative(int i, atomic_t *v);
```
该函数对原子类型的变量v原子地增加i，并判断结果是否为负数，如果是，返回真，否则返回假。
```c
int atomic_add_return(int i, atomic_t *v);
```
该函数对原子类型的变量v原子地增加i，并且返回指向v的指针。
```c
int atomic_sub_return(int i, atomic_t *v);
```
该函数从原子类型的变量v中减去i，并且返回指向v的指针。
```c
int atomic_inc_return(atomic_t * v);
```
该函数对原子类型的变量v原子地增加1并且返回指向v的指针。
```c
int atomic_dec_return(atomic_t * v);
```
***
该函数对原子类型的变量v原子地减1并且返回指向v的指针。
原子操作通常用于实现资源的引用计数，在TCP/IP协议栈的IP碎片处理中，就使用了引用计数，碎片队列结构struct ipq描述了一个IP碎片，字段refcnt就是引用计数器，它的类型为atomic_t，当创建IP碎片时（在函数ip_frag_create中），使用atomic_set函数把它设置为1，当引用该IP碎片时，就使用函数atomic_inc把引用计数加1，当不需要引用该IP碎片时，就使用函数ipq_put来释放该IP碎片，ipq_put使用函数atomic_dec_and_test把引用计数减1并判断引用计数是否为0，如果是就释放IP碎片。函数ipq_kill把IP碎片从ipq队列中删除，并把该删除的IP碎片的引用计数减1（通过使用函数atomic_dec实现）。
***


## 参考:
1. [Linux 内核同步（一）：原子操作](https://stephenzhou.blog.csdn.net/article/details/86597401)
2. [Linux内核中的各种锁](https://blog.csdn.net/godleading/article/details/7825984)


> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=MzAxNzU2MzcwMw==&mid=2649274278&idx=3&sn=5840ff698e3f963c7855d702e842ec47&chksm=83ffbefeb48837e86fed9754986bca6db364a6fe2e2923549a378e8e5dec6e3cf732cdb198e2&scene=0&xtrack=1#rd)

点 击 关 注 上 方＂**两猿社**＂

设 为＂**置 顶 或 星 标**＂，干 货 第 一 时 间 送 达。

![](https://mmbiz.qpic.cn/mmbiz_png/6OkibcrXVmBH2ZO50WrURwTiaNKTH7tCia3VKWA3ibh1dpGhGKXFB2T1eKtfqibUWvQV1j3nczCtPzZF46pamj6dC1w/640?wx_fmt=png)

互 联 网 猿 | 两 猿 社

### 基础知识

#### **RAII**

*   RAII 全称是 “Resource Acquisition is Initialization”，直译过来是 “资源获取即初始化”.
    
*   在构造函数中申请分配资源，在析构函数中释放资源。因为 C++ 的语言机制保证了，当一个对象创建的时候，自动调用构造函数，当对象超出作用域的时候会自动调用析构函数。所以，在 RAII 的指导下，我们应该使用类来管理资源，将资源和对象的生命周期绑定
    
*   RAII 的核心思想是将资源或者状态与对象的生命周期绑定，通过 C++ 的语言机制，实现资源和状态的安全管理, 智能指针是 RAII 最好的例子
    

#### **信号量**

信号量是一种特殊的变量，它只能取自然数值并且只支持两种操作：等待 (P) 和信号(V). 假设有信号量 SV，对其的 P、V 操作如下：

> *   P，如果 SV 的值大于 0，则将其减一；若 SV 的值为 0，则挂起执行
>     
> *   V，如果有其他进行因为等待 SV 而挂起，则唤醒；若没有，则将 SV 值加一
>     

信号量的取值可以是任何自然数，最常用的，最简单的信号量是二进制信号量，只有 0 和 1 两个值.

> *   sem_init 函数用于初始化一个未命名的信号量
>     
> *   sem_destory 函数用于销毁信号量
>     
> *   sem_wait 函数将以原子操作方式将信号量减一, 信号量为 0 时, sem_wait 阻塞
>     
> *   sem_post 函数以原子操作方式将信号量加一, 信号量大于 0 时, 唤醒调用 sem_post 的线程
>     

以上，成功返回 0，失败返回 errno

#### **互斥量**

互斥锁, 也成互斥量, 可以保护关键代码段, 以确保独占式访问. 当进入关键代码段, 获得互斥锁将其加锁; 离开关键代码段, 唤醒等待该互斥锁的线程.

> *   pthread_mutex_init 函数用于初始化互斥锁
>     
> *   pthread_mutex_destory 函数用于销毁互斥锁
>     
> *   pthread_mutex_lock 函数以原子操作方式给互斥锁加锁
>     
> *   pthread_mutex_unlock 函数以原子操作方式给互斥锁解锁
>     

以上，成功返回 0，失败返回 errno

#### **条件变量**

条件变量提供了一种线程间的通知机制, 当某个共享数据达到某个值时, 唤醒等待这个共享数据的线程.

> *   pthread_cond_init 函数用于初始化条件变量
>     
> *   pthread_cond_destory 函数销毁条件变量
>     
> *   pthread_cond_broadcast 函数以广播的方式唤醒**所有**等待目标条件变量的线程
>     
> *   pthread_cond_wait 函数用于等待目标条件变量. 该函数调用时需要传入 **mutex 参数 (加锁的互斥锁)** , 函数执行时, 先把调用线程放入条件变量的请求队列, 然后将互斥锁 mutex 解锁, 当函数成功返回为 0 时, 互斥锁会再次被锁上. **也就是说函数内部会有一次解锁和加锁操作**.
>     

### 功能

#### **锁机制的功能**

*   实现多线程同步，通过锁机制，确保任一时刻只能有一个线程能进入关键代码段.
    

#### **封装的功能**

*   类中主要是 Linux 下三种锁进行封装，将锁的创建于销毁函数封装在类的构造与析构函数中，实现 RAII 机制
    

```
 1class sem{
 2    public:
 3        //构造函数
 4        sem()
 5        {
 6            //信号量初始化
 7            if(sem_init(&m_sem,0,0)!=0){
 8                throw std::exception();
 9            }
10        }
11        //析构函数
12        ~sem()
13        {
14            //信号量销毁
15            sem_destroy(&m_sem);
16        }
17    private:
18        sem_t m_sem;
19};


```

*   将重复使用的代码封装为函数，减少代码的重复，使其更简洁
    

```
 1   //条件变量的使用机制需要配合锁来使用
 2   //内部会有一次加锁和解锁
 3   //封装起来会使得更加简洁
 4   bool wait()
 5   {
 6       int ret=0;
 7       pthread_mutex_lock(&m_mutex);
 8       ret=pthread_cond_wait(&m_cond,&m_mutex);
 9       pthread_mutex_unlock(&m_mutex);
10       return ret==0;
11   }
12   bool signal()
13   {
14       return pthread_cond_signal(&m_cond)==0;
15   }

```

如果本文对你有帮助，`阅读原文`star 一下服务器项目，我们需要你的星星 ^_^.

完。

![](https://mmbiz.qpic.cn/mmbiz_jpg/6OkibcrXVmBH2ZO50WrURwTiaNKTH7tCia3PrKDoB4pKN0Qk32tTqoLWgs7JkOj9QT09dcZZo3xVsMMPiaRwHrPrDg/640?wx_fmt=jpeg)
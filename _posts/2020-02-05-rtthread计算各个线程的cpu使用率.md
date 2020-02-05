---
layout:     post
title:      rtthread计算各个线程的cpu使用率
subtitle:   cpu使用率第二弹
date:       2020-02-05
author:     Hehesheng
header-img: img/67342794_0.jpg
catalog: true
tags:
    - k210
    - 单片机
    - MCU
    - rt-thread
    - risc-v
---

## rt-thread计算cpu使用率

没想到我竟然没鸽！还很快的更新了！

![](https://raw.githubusercontent.com/Hehesheng/blog_img/master/2020/02/05/201102.png)

上一篇文章[计算k210的cpu使用率](https://hehesheng.github.io/2020/02/03/计算k210的cpu使用率/)中说明了计算cpu使用率的方法，这次更进一步，分析各个线程的。

做调试的时候，有时会好奇，这个线程占用了多少cpu资源，但是很可惜，这个功能在rt-thread上没有，官方也没有想出的意思。我这就想起freertos上有这个功能，于是我就想给rt-thread也做个类似的。

### 运行环境

- MAIX-GO （K210双核400M单片机）
- [rt-thread 4.0.3](https://github.com/RT-Thread/rt-thread)
- eclipse GNU

### 基本准备

我首先想的是在尽可能的不改动已有源码的情况下，添加这个功能，于是我就想到可以用系统的时间片进行计时，但是怎么给每个线程添加附加的信息呢？

翻了一下官方的api手册，并没有发现类的功能。不甘心的我去翻看一下rt_thread_t的结构：

```c
struct rt_thread
{
    /* rt object */
    char        name[RT_NAME_MAX];                      /**< the name of thread */
    rt_uint8_t  type;                                   /**< type of object */
    rt_uint8_t  flags;                                  /**< thread's flags */

    rt_list_t   list;                                   /**< the object list */
    rt_list_t   tlist;                                  /**< the thread list */

    /* stack point and entry */
    void       *sp;                                     /**< stack point */
    void       *entry;                                  /**< entry */
    void       *parameter;                              /**< parameter */
    void       *stack_addr;                             /**< stack address */
    rt_uint32_t stack_size;                             /**< stack size */

    /* error code */
    rt_err_t    error;                                  /**< error code */

    rt_uint8_t  stat;                                   /**< thread status */

    rt_uint8_t  bind_cpu;                               /**< thread is bind to cpu */
    rt_uint8_t  oncpu;                                  /**< process on cpu` */

    rt_uint16_t scheduler_lock_nest;                    /**< scheduler lock count */
    rt_uint16_t cpus_lock_nest;                         /**< cpus lock count */
    rt_uint16_t critical_lock_nest;                     /**< critical lock count */

    /* priority */
    rt_uint8_t  current_priority;                       /**< current priority */
    rt_uint8_t  init_priority;                          /**< initialized priority */
    rt_uint32_t number_mask;

    /* thread event */
    rt_uint32_t event_set;
    rt_uint8_t  event_info;

    rt_ubase_t  init_tick;                              /**< thread's initialized tick */
    rt_ubase_t  remaining_tick;                         /**< remaining tick */

    struct rt_timer thread_timer;                       /**< built-in thread timer */

    void (*cleanup)(struct rt_thread *tid);             /**< cleanup function when thread exit */

    rt_uint32_t user_data;                             /**< private user data beyond this thread */
};
```

嗯？有一个user_data成员变量，和绝大部分rtthread中的对象一样。但是为什么是rt_uint32_t类型？不应该是void*吗？我全局搜索了一下，只有rt_thread结构体的user_data是整形变量，可能有遗漏，而且这样在k210这样64位的芯片上会有潜在bug，全局搜索一下，发现调用这个user_data的只有pthread相关的api，那就很好办了，这里**直接将user_data的数据类型改为void\*，并禁用pthread**。

ok，修改完源码，就可以开始了。

### 思路

这里我们计算的思路就是借用user_data加上一些和计时相关的变量，时间精度就用systick。

然后借助线程的调度器钩子，记录线程每次调度之间消耗了多少tick。

因此我们先定义一个结构体

```c
typedef struct thread_usage
{
    rt_uint32_t magic;
    rt_uint32_t enter_tick;
    rt_uint32_t leave_tick;
    rt_uint32_t count_tick;
    rt_uint32_t cost_tick;
} * thread_usage_t;
```

- **magic**变量用于校验结构体是否合法的，这里我们填入一个数字**0x6e6e56e8**

- **enter_tick**记录一次线程进入时的tick
- **leave_tick**记录一次线程退出时的tick
- **count_tick**记录一段时间内，线程总共使用了多少tick
- **cost_tick**用于缓存计算结果

为了能定时更新信息，我们需要一个周期服务的任务，在这里我创建了一个1s一次，优先级为0（最高）的线程负责遍历更新线程信息。

```c
/* 线程服务函数 */
static void update_per_second(void* parm)
{
    struct rt_object_information* info = (struct rt_object_information*)parm;
    thread_usage_t data;
    rt_thread_t tid;
    rt_list_t* node;

    while (1)
    {
        /* 遍历所有线程 */
        for (node = info->object_list.next; node != &(info->object_list); node = node->next)
        {
            tid  = rt_list_entry(node, struct rt_thread, list);
            data = (thread_usage_t)tid->user_data;
            /* 检查线程user_data的合法性 */
            if (data != RT_NULL && data->magic == HOOK_MAGIC_NUM)
            {
                data->cost_tick  = data->count_tick;
                data->count_tick = 0;
            }
        }
        rt_thread_delay(RT_TICK_PER_SECOND);
    }
}
```

至于调度器的钩子，无非就是获取tick，如果user_data为空申请内存，更新数据。

```c
static void schedule_hook(rt_thread_t from, rt_thread_t to)
{
    thread_usage_t from_info, to_info;
    rt_tick_t tick;

    tick = rt_tick_get();
    /* check from data illegal */
    from_info = (thread_usage_t)from->user_data;
    if (from_info == RT_NULL && from->cleanup == RT_NULL)
    {
        from_info = rt_malloc(sizeof(struct thread_usage));
        if (from_info == RT_NULL)
        {
            log_w("No memory");
        }
        else
        {
            from_info->magic      = HOOK_MAGIC_NUM;
            from_info->enter_tick = tick;
            from->user_data       = from_info;
            from->cleanup         = clean_up_call;
        }
    }
    /* user data update */
    if (from_info->magic == HOOK_MAGIC_NUM)
    {
        from_info->leave_tick = tick;
        from_info->count_tick += from_info->leave_tick - from_info->enter_tick;
    }

    /* check to data illegal */
    to_info = (thread_usage_t)to->user_data;
    if (to_info == RT_NULL && to->cleanup == RT_NULL)
    {
        to_info = rt_malloc(sizeof(struct thread_usage));
        if (to_info == RT_NULL)
        {
            log_w("No memory");
        }
        else
        {
            to_info->magic = HOOK_MAGIC_NUM;
            to->user_data  = to_info;
            to->cleanup    = clean_up_call;
        }
    }
    /* user data update */
    if (to_info->magic == HOOK_MAGIC_NUM)
    {
        to_info->enter_tick = tick;
    }
}
```

细心一点就会发现，我们的计算方法使用了额外的内存空间，于是我们借用clean_up函数，这个函数在线程被回收的时候调用，于是我们检查user_data是否为空的时候还要一并检查clean_up是否也为空。

```c
/* clean_up函数 */
static void clean_up_call(rt_thread_t tid)
{
    thread_usage_t info;

    if (tid != RT_NULL && tid->user_data != RT_NULL)
    {
        info = (thread_usage_t)tid->user_data;
        if (info->magic == HOOK_MAGIC_NUM)
        {
            rt_free(info);
        }
    }
}
```

至此，我们计算函数已经全部写好，写一个指令来看看运行效果

```c
static void get_cpu_usage(void)
{
    struct rt_object_information* info = rt_object_get_information(RT_Object_Class_Thread);
    thread_usage_t data;
    rt_thread_t tid;
    rt_list_t* node;
    rt_tick_t cost;

    rt_kprintf("cpu  usage    total\n");
    rt_kprintf("---  ------  ------\n");
    for (int core = 0; core < 2; core++)
    {
        rt_kprintf(" %d   %2d.%02d%%  %6d\n", core, cpus[core].cpu_usage_major, cpus[core].cpu_usage_minor,
                   cpus[core].total_count);
    }
    rt_kprintf("%-*.s  cost\n", RT_NAME_MAX, "thread");
    for (int i = 0; i < RT_NAME_MAX; i++) rt_kprintf("-");
    rt_kprintf(" ------\n");
    for (node = info->object_list.next; node != &(info->object_list); node = node->next)
    {
        tid  = rt_list_entry(node, struct rt_thread, list);
        data = (thread_usage_t)tid->user_data;
        if (data != RT_NULL && data->magic == HOOK_MAGIC_NUM)
        {
            rt_kprintf("%-*.*s %6d\n", RT_NAME_MAX, RT_NAME_MAX, tid->name, data->cost_tick);
        }
        else
        {
            rt_kprintf("%-*.*s   NULL\n", RT_NAME_MAX, RT_NAME_MAX, tid->name);
        }
    }
}
MSH_CMD_EXPORT(get_cpu_usage, get cpu usage);
```

![](https://raw.githubusercontent.com/Hehesheng/blog_img/master/2020/02/06/025648.png)

ok，看来运行正常

### 问题

相信仔细想想的朋友都发现了这种计算的关键问题：时间粒度太大了

什么意思呢，就是如果出现一个任务频繁调度，那么最后计时效果就会严重失真，例如一个线程运行一个很简单的运算，消耗时间基本忽略不计，但是他每一个tick都有调度请求，那么就会出现类似下面的问题：

![](https://raw.githubusercontent.com/Hehesheng/blog_img/master/2020/02/06/030327.png)

这种测占用的方法就是时间片，所有线程时间片和应该接近于一秒钟的tick数乘以核心数，这里的tick和应该为2000，可以看到，这里就测不准了。因为这里的tft线程就是一个频繁调度，但是单次任务简单的线程，他的占用反而为0。

解决方法也十分简单，那就是换粒度更细的时间去计时。

那么就是像freertos中的实现方式类似了，额外配置一个高频率的硬件定时器来完成这个计算事件。

看来这个cpu占用系列还可以更第三期呢，hahahahaha

代码还是更新到[仓库](https://github.com/Hehesheng/k210)里。



觉得有意思就点个star吧，秋梨膏

<img src="https://raw.githubusercontent.com/Hehesheng/blog_img/master/2020/02/06/031452.png" style="zoom:50%;" />
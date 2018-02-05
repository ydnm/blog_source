---
title: Libevent手册-使用events
date: 2017-12-26 01:55:46
categories: Libevent
tags:
---

### 使用 events
-------------------

Libevent的基础执行单位是'event'。每个event代表一组条件，包括:
   
   - 当一个文件描述符已经准备好可读或者可写时
   - 当一个文件描述符将要可读或者可写时(只适用于电平触发的IO复用)
   - 当一个定时器到期触发时
   - 当触发了一个信号时
   - 用户主动触发时

Events拥有相似的生命周期。当你调用一个Libevent函数设置一个事件并将它与一个 event base 关联，它就被 \*初始化\* 了。此时，你可以 'add'，这样这个event就在base中处于 \*待触发状态\*。当event处于待触发状态时，当能够触发这个event的条件发生时(比如，文件描述符状态改变，或者定时器触发)，这个event就变成 \*激活状态\*，并且会运行用户设置的回调函数。如果这个event被设置成 \*永久的\*，它会继续变为待触发状态。如果不是永久的，它运行完回调函数之后就不会再等待被再次触发。你也可以通过'deleting'来停止一个正在等待触发的event。也可以用'add'来让event重新等待触发。

### 构造 event 对象
-------------------

创建一个event, 可以使用event_new()接口。

```
#define EV_TIMEOUT      0x01
#define EV_READ         0x02
#define EV_WRITE        0x04
#define EV_SIGNAL       0x08
#define EV_PERSIST      0x10
#define EV_ET           0x20

typedef void (*event_callback_fn)(evutil_socket_t, short, void *);

struct event *event_new(struct event_base *base, evutil_socket_t fd,
    short what, event_callback_fn cb,
    void *arg);

void event_free(struct event *event);
```

event_new() 函数尝试分配并初始化一个供'base'使用的event。'what'参数是一组上面列出来的枚举组合。(他们的意思在下面说明)。 如果'fd'参数非否，那它就是我们将要检测可读可写事件的文件描
述符。当event被激活，Libevent会调用用户提供的回调函数'cb'，并传递给它参数：文件描述符'fd'，还有一个标记代表event被哪些条件触发，以及回调函数构造时用户传递的'arg'参数。

如果有内部错误或者传参错误，event_new()会返回NULL。

所有新创建的events都是初始化好但是没有进入待触发状态的。为了让一个event进入待触发状态，需要调用event_add()函数。

为了销毁一个event，需要调用event_free()。不管event是待激活状态还是激活状态，调用event_free()都是安全的：这么做可以保证event销毁前是未进入待激活状态并且是未激活的。

```
#include <event2/event.h>

void cb_func(evutil_socket_t fd, short what, void *arg)
{
        const char *data = arg;
        printf("Got an event on socket %d:%s%s%s%s [%s]",
            (int) fd,
            (what&EV_TIMEOUT) ? " timeout" : "",
            (what&EV_READ)    ? " read" : "",
            (what&EV_WRITE)   ? " write" : "",
            (what&EV_SIGNAL)  ? " signal" : "",
            data);
}

void main_loop(evutil_socket_t fd1, evutil_socket_t fd2)
{
        struct event *ev1, *ev2;
        struct timeval five_seconds = {5,0};
        struct event_base *base = event_base_new();

        /* 调用者已设置完成fd1和fd2，并使它们工作在非阻塞模式下。 */

        ev1 = event_new(base, fd1, EV_TIMEOUT|EV_READ|EV_PERSIST, cb_func,
           (char*)"Reading event");
        ev2 = event_new(base, fd2, EV_WRITE|EV_PERSIST, cb_func,
           (char*)"Writing event");

        event_add(ev1, &five_seconds);
        event_add(ev2, NULL);
        event_base_dispatch(base);
}
```

上面的函数都定义在<event2/event.h>中，首次出现在Libevent 2.0.4-alpha中。

### event 触发标记
-------------------

EV_TIMEOUT::
    这个标记表明这个event将会在一段时间延时后会被触发。

    EV_TIMEOUT标记在event被创建初始化时会被忽略：你可以在add这个event的时候设置
    一个延时时间。当延时事件发生时这个标记传递给回调函数的'what'参数中。

EV_READ::
    这个标记表明一个event会在关联的文件描述符上有可读事件时被激活触发。

EV_WRITE::
    这个标记表明一个event会在关联的文件描述符上有可写事件时被激活触发。

EV_SIGNAL::
    用来捕获信号。见"构造信号event"

EV_PERSIST::
    表明这个event是长期的。见 "关于永久event"

EV_ET::
    表明这个event的触发方式为电平触发，并且需要event_base的底层方法支持电平触发。
    这个效果对EV_READ 和 EV_WRITE有效。

自从Libevent 2.0.1-alpha开始，任意数量的events可以在同一时刻被同一个条件触发。举个例子，当一个文件描述符可读时，你可以有2个events被激活，他们的回调函数被执行的顺序是不确定的。

这些标记被定义在<event2/event.h>里。除了EV_ET之外的所有标记都在Libevent 1.0之前的版本里就存在，EV_ET在Libevent 2.0.1-alpha版本中引入。

### 关于永久event
-------------------

通常情况下，当一个event被激活之后(比如它的fd可读可写，又比如它的定时器到期)，它在对应的回调函数被执行前将不再处于新的待激活状态。这样的话，如果你希望让这个event重新处于待激活状态，你就需要在回调函数里再次调用event_add()函数。

但是如果这个event设置了EV_PERSIST标记，那么这个event将是'永久的'。这就意味着这个event被激活执行回调函数之后会继续等待被再次激活。这时候如果你希望让event不再被激活，则需要调用手动调用event_del()。

一个永久event上的定时设置会在event的回调函数运行后被重置。比如，如果你有一个带有
EV_READ|EV_PERSIST标记的event，并且设置了5秒后运行，那么这个event将会被下面条件激活:

  - 当socket可读就绪。
  - 当event距上一次被激活时间达到5秒了。

### 创建一个event作为它自己回调函数的参数
----

通常你可以希望创建一个将自身作为回调函数参数的event。这里你不能通过event_new()的时候直接传递一个指向event的指针作为参数，因为这个时候event还未被创建。为了解决这个总是，你可以使用event_self_cbarg()。

```
void *event_self_cbarg();
```

event_self_cbarg()函数返回一个 "魔法" 指针，当传递给event作为回调函数参数的时候，告诉event_new()创建event的时候将自己作为回调函数的参数。

```
#include <event2/event.h>

static int n_calls = 0;

void cb_func(evutil_socket_t fd, short what, void *arg)
{
    struct event *me = arg;

    printf("cb_func called %d times so far.\n", ++n_calls);

    if (n_calls > 100)
       event_del(me);
}

void run(struct event_base *base)
{
    struct timeval one_sec = { 1, 0 };
    struct event *ev;
    /* We're going to set up a repeating timer to get called called 100
       times. */
    ev = event_new(base, -1, EV_PERSIST, cb_func, event_self_cbarg());
    event_add(ev, &one_sec);
    event_base_dispatch(base);
}
```

这个函数可以被用于event_new(), evtimer_new(), evsignal_new(), event_assign(), evtimer_assign(), 以及 evsignal_assign()等函数。它不能被用于event以外其他回调函数的参数。

event_self_cbarg()函数自Libevent 2.1.1-alpha版本引入。

### 定时器events
---

为了方便，Libevent里面定义了一些以evtimer_开头的宏来代替event_*函数来构造和操作单纯的定时器events。使用这些宏并不会为你带来任何好处，只是能提高你代码的可读性。

```
#define evtimer_new(base, callback, arg) \
    event_new((base), -1, 0, (callback), (arg))
#define evtimer_add(ev, tv) \
    event_add((ev),(tv))
#define evtimer_del(ev) \
    event_del(ev)
#define evtimer_pending(ev, tv_out) \
    event_pending((ev), EV_TIMEOUT, (tv_out))
```

这些宏提出自Libevent 0.6版，除了evtimer_new()函数，它是从Libevent 2.0.1-alpha版
开始出现的。

### 构造信号处理events
---

Libevent也可以监测POSIX风格的信号。为了构造一个信号处理器, 可以使用:

```
#define evsignal_new(base, signum, cb, arg) \
    event_new(base, signum, EV_SIGNAL|EV_PERSIST, cb, arg)
```

参数和event_new差不多，除了我们需要提供一个代表信号的值来取代文件描述符。

```
struct event *hup_event;
struct event_base *base = event_base_new();

/* 收到HUP信号时调用sighup_function函数 */
hup_event = evsignal_new(base, SIGHUP, sighup_function, NULL);
```

注意, 信号回调函数是运行在当信号触发之后的事件循环里的，所以回调函数是安全的，你不用担心回调会在普通的POSIX信号处理函数中被调用。

警告: 不要在一个信号event上设置定时器。暂时是不支持的。

同样的Libevent里也有一系列宏来简化你使用信号events的方式。

```
#define evsignal_add(ev, tv) \
    event_add((ev),(tv))
#define evsignal_del(ev) \
    event_del(ev)
#define evsignal_pending(ev, what, tv_out) \
    event_pending((ev), (what), (tv_out))
```

evsignal_*系列宏出现自Libevent 2.0.1-alpha版本。之前版本中可以使用singal_add()， signal_del()等函数。

### 使用singal时需要注意的点
---

在Libevent当前的版本中，大部分底层方法，一个进程中同一时刻只能使用一个event_base来监听信号。如果你同时向2个event_base中添加信号events， ---甚至用的是2个不同的信号!--- 只有一个event_base可以收到信号。

但是底层用kqueue的话没有这个限制。

### 不使用event_new()构建events。
--- 

为了性能和一些其他原因，有人希望可以将event作为一个较大的结构体的成员变量。跟使用单个event想比，可以节约下面几点:
   
   - 用new创建小结构体带来的内存碎片消耗。
   - 用event指针取值带来的时间消耗。
   - 缓存未命中带来的时间消耗。

使用这些函数会打破与其他版本Libevent的二进制兼容，有可能会造成event的大小不一致。

其实原来用event_new创建event的方式只有_非常_小的消耗，对于大部分应用来说无关紧要。你最好使用event_new()来创建event，除非你*明确*你正在遭受由使用event_new()带来的严重性能问题。如果将event作为结构体成员变量，在不同版本中使用event_assign()可能会导致一些隐性错误。

```
int event_assign(struct event *event, struct event_base *base,
    evutil_socket_t fd, short what,
    void (*callback)(evutil_socket_t, short, void *), void *arg);
```

event_assign()的大部分参数和event_new()一样，除了'event'参数，它是一个指向未初始化的event的指针。调用成功返回0，失败或者参数错误返回-1。

```
#include <event2/event.h>
/* Watch out!  Including event_struct.h means that your code will not
 * be binary-compatible with future versions of Libevent. */
/* 注意！ 包含event_struct.h头文件意味着你的代码将不能和
 * 未来版本的Libevent二进制兼容。 */
#include <event2/event_struct.h>
#include <stdlib.h>

struct event_pair {
         evutil_socket_t fd;
         struct event read_event;
         struct event write_event;
};
void readcb(evutil_socket_t, short, void *);
void writecb(evutil_socket_t, short, void *);
struct event_pair *event_pair_new(struct event_base *base, evutil_socket_t fd)
{
        struct event_pair *p = malloc(sizeof(struct event_pair));
        if (!p) return NULL;
        p->fd = fd;
        event_assign(&p->read_event, base, fd, EV_READ|EV_PERSIST, readcb, p);
        event_assign(&p->write_event, base, fd, EV_WRITE|EV_PERSIST, writecb, p);
        return p;
}
```

你也可以用event_assign()来初始化栈上创建的以及静态的event结构体。

.警告
绝对不要对一个已经追加到event_base中处于待激活状态的event使用event_assign()函数。这么做会导致非常难以察觉的错误。如果event已经初始化完成并且处于待激活状态，在你在这个event上使用event_assign() \*之前\* 先使用 event_del() 来删除它。

下面有一些宏来方便你使用event_assign()来初始化一个定时器或信号event:

```
#define evtimer_assign(event, base, callback, arg) \
    event_assign(event, base, -1, 0, callback, arg)
#define evsignal_assign(event, base, signum, callback, arg) \
    event_assign(event, base, signum, EV_SIGNAL|EV_PERSIST, callback, arg)
```

如果你需要使用event_assign() *并且* 需要与未来版本保持二进制兼容，那么你可以运行时向Libevent库请求'event结构体'的具体大小:

```
size_t event_get_struct_event_size(void);
```

这个函数返回需要为event预留内存大小的字节数。跟之前一样，你需要明确知道堆构建event是造成你程序里严重问题根源的时候再去使用它，因为它会把你代码的可读性变差。

注意event_get_struct_event_size()可能会在未来版本获得比'sizeof(struct event)' _小_ 的数值。如果发生这种情况，意味着'event'结构体最后的那些额外字节是为了给未来版本的Libevent做保留的填充字节。

这里有一个和上面一样的例子，但是用event_get_struct_size()获取运行时event大小，来取代 event_struct.h 头文件中 event 结构体的大小。

```
#include <event2/event.h>
#include <stdlib.h>

/* 当我们内存中申请一个event_pair结构，我们实际上会在结构体后面
 * 申请更多的内存来保存event。我们定义一些宏
 * 来不容易出错地访问这些event。*/
struct event_pair {
         evutil_socket_t fd;
};

/* Macro: yield the struct event 'offset' bytes from the start of 'p' */
/* 宏: 返回指针'p'偏移'offset'字节的地址作为event指针。 */
#define EVENT_AT_OFFSET(p, offset) \
            ((struct event*) ( ((char*)(p)) + (offset) ))
/* Macro: yield the read event of an event_pair */
/* 宏: 返回event_pair中读事件event结构的地址 */
#define READEV_PTR(pair) \
            EVENT_AT_OFFSET((pair), sizeof(struct event_pair))
/* Macro: yield the write event of an event_pair */
/* 宏: 返回event_pair中写事件event结构的地址 */
#define WRITEEV_PTR(pair) \
            EVENT_AT_OFFSET((pair), \
                sizeof(struct event_pair)+event_get_struct_event_size())

/* Macro: yield the actual size to allocate for an event_pair */
/* 宏: 返回为event_pair结构体申请内存的实际大小 */
#define EVENT_PAIR_SIZE() \
            (sizeof(struct event_pair)+2*event_get_struct_event_size())

void readcb(evutil_socket_t, short, void *);
void writecb(evutil_socket_t, short, void *);
struct event_pair *event_pair_new(struct event_base *base, evutil_socket_t fd)
{
        struct event_pair *p = malloc(EVENT_PAIR_SIZE());
        if (!p) return NULL;
        p->fd = fd;
        event_assign(READEV_PTR(p), base, fd, EV_READ|EV_PERSIST, readcb, p);
        event_assign(WRITEEV_PTR(p), base, fd, EV_WRITE|EV_PERSIST, writecb, p);
        return p;
}
```

event_assign()函数字义在<event2/event.h>头文件中。出现自Libevent 2.0.1-alpha版本。2.0.3-alpha版本开始，它返回一个int类型。之前没有返回值。event_get_struct_event_size() 函数始于Libevent 2.0.4-alpha版本。event结构体是定义在<event2/event_struct.h>头文件中的。

### 设置events等待激活和取消等待激活
--- 

一旦你构造了一个event，它实际上还不能做任何事，除非你将它add到event_base中使它处于待激活状态。你可以使用event_add来这么做:

```
int event_add(struct event *ev, const struct timeval *tv);
```

对一个未处于等待激活状态的event调用event_add会让它加入到配置了的event_base中。函数成功时返回0，失败返回-1。如果'tv'为空，这个event将不会设置超时。相反的，'tv'包含超时设置的秒数和毫秒数。

如果你对一个 \*已经\* 处于等激活状态的event调用 event_add()，不会改变event的状态，并且如果给了超时的话会重置超时时间。如果event已经在等待激活，并且你用超时时间参数为空来重新add它，event_add() 函数不会有任何效果。

注意：不要将参数'tv'设置成你希望程序运行的时间。比如你在2010年1月1号用"tv->tv_sec = time(NULL)+10;", 那么你的超时将会在40年后过期，而非10秒后过期。

```
int event_del(struct event *ev);
```

对一个初始化好的event调用event_del可以使它取消等待激活状态。如果event并未正在等待或者已被激活，这么做没有任何效果。函数成功时返回0，失败返回-1。

注意：如果你在一个event激活后且它的回调函数被执行前删除一个event，那么它的回调函数将不会被执行。

```
int event_remove_timer(struct event *ev);
```

最后，使用event_remove_timer可以让你在不影响event中信号或者IO组件的情况下，移除一个已处于等待激活的event的超时设定。如果event没有设置超时， event_remove_timer()函数没有任何效果。如果event只有超时设定而没有IO或者信号组件，那么event_remove_timer和event_del的效果一样。函数成功时返回0，失败时返回-1。

这些函数被定义在<event2/event.h>头文件中，event_add() 和 event_del()出现自Libevent 0.1版。event_remove_timer()是在2.1.2-alpha版中新增加的。

### 事件的优先级
--- 

当有多个event在同一时刻触发，Libevent不会去定义他们的回调函数被执行的先后顺序。你可以通过定义优先级来让一些event比其他的event更重要。

在之前一个章节的讨论过，每个 event_base 关联了一个或者多个优先级。在添加一个event到 event_base 之前，在event初始化之后，你可以设置它的优先级。

```
int event_priority_set(struct event *event, int priority);
```

event的优先级是一个从0到event_base优先级总数-1的数值。这个函数成功时返回0，失败返回-1。

当拥有多个优先级的多个event被同时激活时，低优先级的event不会马上运行。而Libevent会运行高优先级的
event，然后再次检查event。直到没有高优先级的event激活时才会运行低优先级的event。

```
#include <event2/event.h>

void read_cb(evutil_socket_t, short, void *);
void write_cb(evutil_socket_t, short, void *);

void main_loop(evutil_socket_t fd)
{
  struct event *important, *unimportant;
  struct event_base *base;

  base = event_base_new();
  event_base_priority_init(base, 2);
  /* 现在base有优先级0和1 */
  important = event_new(base, fd, EV_WRITE|EV_PERSIST, write_cb, NULL);
  unimportant = event_new(base, fd, EV_READ|EV_PERSIST, read_cb, NULL);
  event_priority_set(important, 0);
  event_priority_set(unimportant, 1);

  /* 现在，只要fd准备可写的时候，写数据的回调函数就会在读数据之前被执行。而读数据不会被执行，除非写事件不再被激活。*/
}
```

TODO
当你没有设置event的优先级的时候，它的默认优先级是event_base中优先级队列总数除以2。

这个函数被定义在<event2/event.h>头文件里，首次出现自Libevent 1.0版。

### 检查事件状态
--- 

有时候你你可能需要知道一个event是否已被添加,并且检查它绑定了什么事件。

```
int event_pending(const struct event *ev, short what, struct timeval *tv_out);

#define event_get_signal(ev)
evutil_socket_t event_get_fd(const struct event *ev);
struct event_base *event_get_base(const struct event *ev);
short event_get_events(const struct event *ev);
event_callback_fn event_get_callback(const struct event *ev);
void *event_get_callback_arg(const struct event *ev);
int event_get_priority(const struct event *ev);

void event_get_assignment(const struct event *event,
        struct event_base **base_out,
        evutil_socket_t *fd_out,
        short *events_out,
        event_callback_fn *callback_out,
        void **arg_out);
```

event_pending函数能够确定给定的event是否正在等待激活或者已经被激活。如果是，并且EV_READ, EV_WRITE, EV_SIGNAL, 和 EV_TIMEOUT等任意标记被设置到参数'what'中，函数会返回一个数，表示event是否正在等待其中的某几个条件或者已被某几个条件激活，这个数是这些条件的按位或。如果提供了'tv_out'参数，并且标记EV_TIMEOUT被设置到'what'参数中，那么'tv_out'参数会被设置成event还有多少时间触发。

event_get_fd() 和 event_get_signal() 函数返回一个event中设置的文件描述符或信号值。event_get_base() 函数返回event添加到的那个event_base。event_get_events()函数返回event设置监听的事件类型标记(EV_READ, EV_WRITE等)。event_get_callback() 和 event_get_callback_arg() 函数返回event的回调函数和自定义参数指针。event_get_priority() 函数返回event当前分配的优先级。

event_get_assignment() 函数拷贝event参数里的所有字段到该函数给定参数里的指针上。如果哪个参数没有提供指针，那么那个字段会被忽略。

```
#include <event2/event.h>
#include <stdio.h>

/* 改变参数'ev'的回调函数和回调函数参数, 此时'ev'应该还不能开始监听
 * 事件等待激活。 */
int replace_callback(struct event *ev, event_callback_fn new_callback,
    void *new_callback_arg)
{
    struct event_base *base;
    evutil_socket_t fd;
    short events;

    int pending;

    pending = event_pending(ev, EV_READ|EV_WRITE|EV_SIGNAL|EV_TIMEOUT,NULL);
    if (pending) {
        /* 我们不应该到达这里因为我们不能重新分配一个已经处于等待激活状态
         * 的event。这是很不好的。*/
        fprintf(stderr,
                "Error! replace_callback called on a pending event!\n");
        return -1;
    }

    event_get_assignment(ev, &base, &fd, &events,
                         NULL /* ignore old callback */ ,
                         NULL /* ignore old callback argument */);

    event_assign(ev, base, fd, events, new_callback, new_callback_arg);
    return 0;
}
```

这些函数被定义在<event2/event.h>头文件中。event_pending() 函数出现自Libevent 0.1版。Libevent 2.0.1-alpha版引进了event_get_fd() 和 event_get_signal()。Libevent 2.0.2-alpha版引进了event_get_base()。Libevent 2.1.2-alpha版添加了
event_get_priority()。其他函数新出现在Libevent 2.0.4-alpha版本中。

### 查看当前正在运行的event
--- 

为了调试或者其他目的，你可能希望获得当前正在运行的event的指针。

```
struct event *event_base_get_running_event(struct event_base *base);
```

需要注意的是这个函数只能作用于提供的event_base的主循环里，在其他地方或者另外的线程调用都不行，会带来未知错误。

这个函数定义在<event2/event.h>头文件中。自Libevent 2.1.1-alpha版本引入。

### 设置一次性event
---

如果你不需要多次添加一个event，或者添加完之后马上要删除它，并且它不是永久的，那你可以使用event_base_once()。

```
int event_base_once(struct event_base *, evutil_socket_t, short,
  void (*)(evutil_socket_t, short, void *), void *, const struct timeval *);
```

这个函数接口跟 event_new() 一样，除了它不支持EV_SIGNAL和EV_PERSIST。这个计划中的event会以默认的优先级插入和运行。当它的回调函数运行完成之后，Libevent会自动释放这个内部的event结构体。这个函数成功时返回0，失败返回-1。

通过event_base_once添加的event不能被删除或者手动触发: 如果你希望能够删除一个event，请使用通常的 event_new() 或 event_assign() 接口。

注意在Libevent 2.0版之前，如果一个一次性event从来没有被激活过，那这个内部event将永远不会从内存上释放。从Libevent 2.1.2-alpha开始，这些event的内存会在event_base被释放的时候被释放。

### 手动激活一个event
---

少部分情况下，你可能希望在一个event的条件未发生的情况下手动激活一个event。

```
void event_active(struct event *ev, int what, short ncalls);
```

这个函数可以使'ev'参数指向的event通过'what'参数的标记激活，(EV_READ, EV_WRITE 和 EV_TIMEOUT的组合)。这个event不需要提前设置成等待激活状态，手动激活它也不会让他变成等待激活状态。

警告: 在一个event上递归地调用event_active()函数会导致资源耗尽。下面的代码片断是一个不正确使用event_active的坏例子。

.坏例子：使用event_active()产生死循环。
```
struct event *ev;

static void cb(int sock, short which, void *arg) {
        /* 哎呀: 没有条件得在event回调函数内部对同一个event调用event_active意味着其他event不会有机会运行*/

	event_active(ev, EV_WRITE, 0);
}

int main(int argc, char **argv) {
	struct event_base *base = event_base_new();

	ev = event_new(base, -1, EV_PERSIST | EV_READ, cb, NULL);

	event_add(ev, NULL);

	event_active(ev, EV_WRITE, 0);

	event_base_loop(base, 0);

	return 0;
}
```

这样用会产生一个问题，一个event循环只执行了一次但是会无限循环调用'cb'回调函数。

//BUILD: INC:event2/event.h
.例子：使用定时器解决上面的问题是可选方案之一
```

struct event *ev;
struct timeval tv;

static void cb(int sock, short which, void *arg) {
   if (!evtimer_pending(ev, NULL)) {
       event_del(ev);
       evtimer_add(ev, &tv);
   }
}

int main(int argc, char **argv) {
   struct event_base *base = event_base_new();

   tv.tv_sec = 0;
   tv.tv_usec = 0;

   ev = evtimer_new(base, cb, NULL);

   evtimer_add(ev, &tv);

   event_base_loop(base, 0);

   return 0;
}
```

//BUILD: INC:event2/event.h
.例子: 使用event_config_set_max_dispatch_interval()作为另一种可行的解决方案。
```
struct event *ev;

static void cb(int sock, short which, void *arg) {
	event_active(ev, EV_WRITE, 0);
}

int main(int argc, char **argv) {
        struct event_config *cfg = event_config_new();
        /* 在检查一个event之前最多允许运行16次回调函数 */
        event_config_set_max_dispatch_interval(cfg, NULL, 16, 0);
	struct event_base *base = event_base_new_with_config(cfg);
	ev = event_new(base, -1, EV_PERSIST | EV_READ, cb, NULL);

	event_add(ev, NULL);

	event_active(ev, EV_WRITE, 0);

	event_base_loop(base, 0);

	return 0;
}
```

这个函数定义在<event2/event.h>头文件中。出现自Libevent 0.3版本。

### 优化相同延时定时器
----

当前Libevent版本使用二叉堆算法来王跟踪处于等待激活状态的event的超时时间。二叉堆为排序提供了较高的性能，以及复杂度为O(lg n)的插入删除操作。如果你用来插入延时为随机分布的event，这是最佳方案，但是如果你有大量的event使用相同的延时，那就不太合适了。

举个例子，假设你有一万个event，每个都会在被添加到event_base之后5秒的时候被触发。对于这种需要的解决方案，你可以使用复杂度为O(1)的双端队列来实现。

当然，你并不希望使用一个队列来做为你所有定时器的实现，毕竟队列只在所有延时都是固定值的时候才更快。如果一些延时是或多或少随机分布的，这样添加一个这样的延时到队列的时候需要消耗O(n)的时间，这就意味着比二叉堆更糟糕了。

Libevent可以让你通过替换一些定时器使用队列，另一些使用二叉堆的方式解决这个问题。为了这么做，你需要向Libevent申请一个特殊的"通用延时定时器"，然后就可以用来添加定时event了。如果你有大量固定延时时间来激活的event，这么优化可以提升定时器性能。

```
const struct timeval *event_base_init_common_timeout(
    struct event_base *base, const struct timeval *duration);
```

这个函数需要一个event_base和一个固定延时duration作为参数进行初始化，它返回一个特殊的timeval结构体指针，你可以用它来指定一个event应当被添加到O(1)复杂度的列表中，而不是O(lg n)复杂度的二叉堆。这个特殊的timeval在你的代码里可以被自由拷贝或者重新分配。它只作用于你用来构造它的指定的event_base中。绝对不要使用它的真实值：因为Libevent用他们来确定使用哪个队列。

```
#include <event2/event.h>
#include <string.h>

/* 我们将会在一个给定的event_base上创建大量定时10秒触发的event，
 * 如果initialize_timeout函数被调用，我们会告诉Libevent添加这些event到
 * 复杂度为O(1)的队列中去。 */
struct timeval ten_seconds = { 10, 0 };

void initialize_timeout(struct event_base *base)
{
    struct timeval tv_in = { 10, 0 };
    const struct timeval *tv_out;
    tv_out = event_base_init_common_timeout(base, &tv_in);
    memcpy(&ten_seconds, tv_out, sizeof(struct timeval));
}

int my_event_add(struct event *ev, const struct timeval *tv)
{
    /* 注意这个event必须添加到我们传给initialize_timeout的同一个event_base中 */
    if (tv && tv->tv_sec == 10 && tv->tv_usec == 0)
        return event_add(ev, &ten_seconds);
    else
        return event_add(ev, tv);
}
```

如同所有优化函数，你应当尽量避免使用通用延时定时器相关功能，除非你明确知道这么做对你是有益的。

这些函数引进自Libevent 2.0.4-alpha版本。

### 判断一个event是否健康，避免其内存已被清除
---

Libevent提供了一些函数可以帮忙你分辨是否是已初始化过的event，避免它的内存已被清除了。(比如，使用calloc()来申请内存或者使用memset()和bzero()修改过内存)。

```
int event_initialized(const struct event *ev);

#define evsignal_initialized(ev) event_initialized(ev)
#define evtimer_initialized(ev) event_initialized(ev)
```

.警告
这些函数不能可靠地分辨已经初始化的event和一块未初始化的内存。你不应该使用它们，除非你明确那块内存是已经被清除了的或者已被初始化为event了的。

通常情况，你不需要使用这些函数，除非你考虑到一些非常特殊的应用。那些event_new函数返回的event始终是初始化过的。

```
#include <event2/event.h>
#include <stdlib.h>

struct reader {
    evutil_socket_t fd;
};

#define READER_ACTUAL_SIZE() \
    (sizeof(struct reader) + \
     event_get_struct_event_size())

#define READER_EVENT_PTR(r) \
    ((struct event *) (((char*)(r))+sizeof(struct reader)))

struct reader *allocate_reader(evutil_socket_t fd)
{
    struct reader *r = calloc(1, READER_ACTUAL_SIZE());
    if (r)
        r->fd = fd;
    return r;
}

void readcb(evutil_socket_t, short, void *);
int add_reader(struct reader *r, struct event_base *b)
{
    struct event *ev = READER_EVENT_PTR(r);
    if (!event_initialized(ev))
        event_assign(ev, b, r->fd, EV_READ, readcb, r);
    return event_add(ev, NULL);
}
```

event_initialized()函数出现自Libevent 0.3版。

### 废弃的event操作函数
---

在Libevent 2.0版本之前是没有event_assign()和event_new()的。你可以用event_set() 来代替，用来关联event和当前event_base。如果你需要使用多于一个的event_base， 你需要记住在它后面调用event_base_set()来确保event被关联到你希望使用的event_base
上去。

```
void event_set(struct event *event, evutil_socket_t fd, short what,
        void(*callback)(evutil_socket_t, short, void *), void *arg);
int event_base_set(struct event_base *base, struct event *event);
```

event_set()函数和event_assign()很像，除了这只能用于当前event base的用法。 event_base_set()函数改变关联到event上的event_base。

这里还有一个event_set()的变种，处理定时器和信号的时候会更方便：evtimer_set() 是粗糙版evtimer_assign()。同样的evsignal_set()是粗糙版evsignal_assign()。

Libevent 2.0之前的版本使用"signal_"来做为信号相关变体函数的前缀，比如signal_set() 等，而不是用"evsignal_"。(这就是说，原来有signal_set(), signal_add(), signal_del(), signal_pending(), 和 signal_initialized())。更古老的Libevent版本(0.6之前) 使用"timeout_"前缀来代替"evtimer_"。因此如果你在使用这些古老的版本，你可能会见到 timeout_add(), timeout_del(),timeout_initialized(), timeout_set(), timeout_pending()
等函数。

Libevent老版本(2.0之前)使用2个宏EVENT_FD() 和 EVENT_SIGNAL()来取代event_get_fd() 和 event_get_signal() 函数。这些宏会直接检查event结构体内容，所以它们造成了版本之间的二进制不兼容。在2.0之后的版本中，这些宏只是event_get_fd() 和 event_get_signal()
的一个别名而已。

因为在Libevent 2.0之前的版本不支持锁，所以在运行event_base线程之外的线程运行任何能改动event状态的函数都是不安全的。这些函数包括event_add(), event_del(), event_active(), 和 event_base_once()等。

这里还有一个event_once()函数来扮演event_base_once()的角色，但是仅能用于当前event_base。

EV_PERSIST标记在Libevent 2.0之前的版本不能很好的和定时器一起工作。无论何时event激活时，本应该重置event的定时时间，但是EV_PERSIST标记什么都不会做。

Libevent 2.0之前的版本不支持同时插入绑定同个文件描述符fd且监听相同读写事件的多个event。换句话说，每个文件描述符fd上同一时间只有一个event可以等待可读就绪。同一时间也只有一个event可以等待可写事件。


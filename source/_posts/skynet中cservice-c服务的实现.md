---
title: skynet中cservice c服务的实现
date: 2017-08-05 04:38:38
categories: 框架
tags: skynet
---

很早之前就有写博客的念头，无奈始终败给了惰性。随着工作时间的增长，发现记性越来越不及从前。可能是有些东西，工作中不是经常用到，但是出问题需要去Google查的偏偏是这些东西。遇到一次，查完过了很久又遇到，又要再去查，感觉真是蠢爆了。所以这次无论如何也要开始写博客，好记性不如烂笔头。即使以后再遇到同样的问题，自己写过一遍后，印象总会更深刻一点。

第一篇博客写什么内容，考虑了很久。还是决定写点关于skynet的东西。从13年开始接触skynet，期间反复读过源码，但是还是因为惰性，始终没有做笔记。读源码的过程中，我不想记录，因为怕思路被打断。读完之后，还是不想记录，因为觉得自己已经懂了。这种想法是愚蠢的，最不能要的。

引用一句[skynet](https://github.com/cloudwu/skynet)项目[wiki](https://github.com/cloudwu/skynet/wiki)上的介绍：
> skynet 是一个轻量级的为在线游戏服务器打造的框架。

wiki上的内容已经很完善了，一个开源项目离不开社区的支持反馈，当然作者[云风](https://baike.baidu.com/item/%E4%BA%91%E9%A3%8E/20472324)的作用才是最重要的。

skynet中，以服务为单位去处理逻辑。而服务包括C服务和Lua服务。这次先来记录一下C服务的实现。以源码自带的logger服务为例。skynet中C服务是以动态库的形势编译，存放在cservice目录下。logger服务的源码文件为：[service-src/service_logger.c](https://github.com/cloudwu/skynet/blob/master/service-src/service_logger.c)，编译skynet的时候，会将logger服务编译成logger.so。详情参考[Makefile](https://github.com/cloudwu/skynet/blob/master/Makefile)。

service_logger.c源码中，主要看一个数据结构和4个方法：

```
struct logger {
	FILE * handle;
	char * filename;
	int close;
};

// create
struct logger * logger_create(void);

// init
int logger_init(struct logger * inst, struct skynet_context *ctx, const char * parm);

// release
void logger_release(struct logger * inst);

// callback
static int logger_cb(struct skynet_context * context, void *ud, int type, int session, uint32_t source, const void * msg, size_t sz);

```

一个C服务，基本就需要几个方法：

```
(module_name)_create
(module_name)_init
(module_name)_release
(module_name)_signal
```
其中(module\_name)_signal在logger中并没有用到，所以没实现。下面来找找logger服务初始化的地方。

skynet框架启动的入口是[skynet-src/skynet_start.c](https://github.com/cloudwu/skynet/blob/master/skynet-src/skynet_start.c)文件中的: 

`void skynet_start(struct skynet_config * config)`

初始化logger的地方是skynet_start函数中：

`struct skynet_context *ctx = skynet_context_new(config->logservice, config->logger);`

skynet\_context_new函数的作用是创建并初始化一个服务。其中关键的几个调用是：

```
skynet_module_query(name);
skynet_module_instance_create(mod);
skynet_module_instance_init(mod, inst, ctx, param);

```
这3个方法分别是用来查询，创建，初始化服务的。在文件[skynet-src/skynet_module.c](https://github.com/cloudwu/skynet/blob/master/skynet-src/skynet_module.c)中，其中create和init会分别调用对应服务的module\_create方法和module\_init方法。在logger服务中就是logger\_create()和logger_init()。另外skynet\_module中还有

```
void skynet_module_instance_release(struct skynet_module *m, void *inst);

void skynet_module_instance_signal(struct skynet_module *m, void *inst, int signal);
```
两个方法，对应着module\_release()和module_signal()。

具体实现参考[skynet-src/skynet_module.c](https://github.com/cloudwu/skynet/blob/master/skynet-src/skynet_module.c)，简单点说是skynet\_module_query用传进去的模块名，去查找是否有已加载了的服务动态库。没有的话，则用`dlopen`尝试加载。如果加载成功则将其缓存并返回其句柄。过程中还用`dlsym`调用查看是动态库中否有模块名+"\_create | init | release | signal"这几个函数，如果有，刚将其函数地址赋值给skynet\_module数据结构中对应的函数指针，达到映射的作用。这样，在调skynet\_module\_instance\_XXX这几个函数的时候，能够调用到对应的module\_XXX()函数。下面是man手册中对`dlopen`和`dlsym`的描述：

> dlopen: load and link a dynamic library or bundle

> dlsym: get address of a symbol

在logger服务的logger\_init()方法中，根据参数，可以设置输出日志到具体文件还是标准输出。如果没有带参数，则输出到标准输出。然后调用skynet\_command()注册具名服务logger，还有skynet\_callback()方法，注册回调函数到logger_cb()，这样当框架内有发给logger服务的消息的时候，就会进入logger\_cg函数进行处理，输出到文件或者标准输出。这样一个logger服务就完成了。

至于具体的服务创建过程中的消息队列创建和消息分发，会在后续博文中介绍。

以上内容纯属个人理解，水平有限，有很大可能理解偏差，欢迎指正：）mail:xcjlanqiu@gmail.com

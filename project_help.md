+ http://download.redis.io/releases/redis-5.0.8.tar.gz

+ mac + CLion + redis5 本地调试/运行:<https://www.jianshu.com/p/904b44911ab9>
    - 要进入src目录执行`sh mkreleasehdr.sh`
    - 根目录cmake . && make
    - 运行redis-server

+ 更改clion，cmake版本
   - 更改preference, Toolchains, 
   - which cmake : `/usr/local/bin/cmake`
   
+ crtl +option + H 查看调用链（看设置keymap）   
   

### 源码

+ Redis源码从哪里读起？:<http://zhangtielei.com/posts/blog-redis-how-to-start.html>
main方法：int main(int argc, char **argv) （server.c）
main函数开始执行后的逻辑可以分为两个阶段：

各种初始化（包括事件循环的初始化）；
执行事件循环。

<pre>
配置加载和初始化。
创建事件循环。
开始socket监听。
注册timer事件回调。
注册I/O事件回调。
初始化后台线程。
启动事件循环。
</pre>


关于事件机制，还有一些信息值得关注：业界已经有一些比较成熟的开源的事件库了，典型的比如libevent[20]和libev[21]。一般来说，这些开源库屏蔽了非常复杂的底层系统细节，并对不同的系统版本实现做了兼容，是非常有价值的。那为什么Redis的作者还是自己实现了一套呢？在Google Group的一个帖子上，Redis的作者给出了一些原因。帖子地址如下：

https://groups.google.com/group/redis-db/browse_thread/thread/b52814e9ef15b8d0/
原因大致总结起来就是：

不想引入太大的外部依赖。比如libevent太大了，比Redis的代码库还大。
方便做一些定制化的开发。
第三方库有时候会出现一些意想不到的bug。

---

1. 渐进式rehash（搜"如果字典处于搬迁过程中，要将新的元素挂接到新的数组下面"） ；
databasesCron （定时任务rehash）

美团针对Redis Rehash机制的探索和实践：<https://mp.weixin.qq.com/s/ufoLJiXE0wU4Bc7ZbE9cDQ>

2. 定时任务
server.c -> initServer
```C
    /* Create the timer callback, this is our way to process many background
     * operations incrementally, like clients timeout, eviction of unaccessed
     * expired keys and so forth. */
    if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        serverPanic("Can't create event loop timers.");
        exit(1);
    }
```
将定时执行的函数注册到aeEventLoop（server.el），链表结构（eventLoop->timeEventHead = te）
定时任务执行processTimeEvents（ae.c）
无限循环检查定时任务
```C
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```
比对当前时间判断是否需要执行
```C
/* How many milliseconds we need to wait for the next
             * time event to fire? */
            long long ms =
                (shortest->when_sec - now_sec)*1000 +
                shortest->when_ms - now_ms;
```
gettimeofday(&tv, NULL);

定时器：<https://blog.csdn.net/wenfh2020/article/details/106199226>


3. Redis事件循环器(AE)实现剖析
+ <https://zhuanlan.zhihu.com/p/92739237>
aeCreateEventLoop
 maxclients代表用户配置的最大连接数，可在启动时由--maxclients指定，默认为10000。
aeCreateFileEvent表示将某个fd的某些事件注册到eventloop中。
 AE_READABLE 可读事件
 AE_WRITABLE 可写事件
 AE_BARRIER 该事件稍后在"事件的等待和处理" 

int listenToPort(int port, int *fds, int *count)  (server.c)
监听端口，获取描述fd



    
    
    
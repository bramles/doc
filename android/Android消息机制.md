### java层

```c
//相关源码
framework/base/core/java/andorid/os/
  - Handler.java
  - Looper.java
  - Message.java
  - MessageQueue.java
```

![class](../img/handler_class.jpg)

**消息分发流程**

- Handler通过sendMessage()发送Message到MessageQueue队列；
- Looper通过loop()，不断提取出达到触发条件的Message，并将Message交给target来处理；
- 经过dispatchMessage()后，交回给Handler的handleMessage()来进行相应地处理。
- 将Message加入MessageQueue时，会往管道写入字符，唤醒loop线程；如果MessageQueue中没有Message，并处于Idle状态，则会执行IdelHandler接口中的方法，往往用于做一些清理性地工作。

![handler](../img/handler_java.jpg)

**消息分发的优先级：**

1. Message的回调方法：`message.callback.run()`，优先级最高；
2. Handler的回调方法：`Handler.mCallback.handleMessage(msg)`，优先级仅次于1；
3. Handler的默认方法：`Handler.handleMessage(msg)`，优先级最低。

**消息缓存：**

为了提供效率，提供了一个大小为50的Message缓存队列，减少对象不断创建与销毁的过程。



#### native层

```
//java层的消息队列
framework/base/core/java/andorid/os/MessageQueue.java
//native层消息队列，连接native和java层
framework/base/core/jni/android_os_MessageQueue.cpp
framework/base/core/java/andorid/os/Looper.java
system/core/libutils/Looper.cpp
system/core/include/utils/Looper.h
system/core/libutils/RefBase.cpp
framework/base/native/android/looper.cpp
framework/native/include/android/looper.h
```

![native](/Users/liujunjiang/doc/img/native_handler.png)

**获取消息流程**

1. 先调用epoll_wait()，这是阻塞方法，用于等待事件发生或者超时；
2. 对于epoll_wait()返回，当且仅当以下3种情况出现：
   - POLL_ERROR，发生错误，直接跳转到Done；
   - POLL_TIMEOUT，发生超时，直接跳转到Done；
   - 检测到管道有事件发生，则再根据情况做相应处理：
     - 如果是管道读端产生事件，则直接读取管道的数据；
     - 如果是其他事件，则处理request，生成对应的reponse对象，push到reponse数组；
3. 进入Done标记位的代码段：
   - 先处理Native的Message，调用Native 的Handler来处理该Message;
   - 再处理Response数组，POLL_CALLBACK类型的事件；

![poll](/Users/liujunjiang/doc/img/poll_once.png)

**阻塞的实现：**

```java
Message next() {
    //native层消息队列对象的地址
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    //阻塞时间
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
		//阻塞方法，等待底层或上层消息插入队列或者超时唤醒
        //这个方法会默认先处理native层消息队列中的消息
        //-1：没有消息时，等待时间为-1，无限等待
        //0：非阻塞
        //0++:阻塞固定时间后唤醒
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            ...
            if (msg != null) {
                if (now < msg.when) {
                    //队列中第一个需要执行的时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
					...                
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }
            ...
        }
		//存在消息时，不阻塞
        nextPollTimeoutMillis = 0;
    }
}
```

1.android 6.0以前，epoll监听管道

2.android 6.0以后，使用event_fd

**native层总结**

- nativeInit()方法：
  - 创建了NativeMessageQueue对象，增加其引用计数，并将NativeMessageQueue指针mPtr保存在Java层的MessageQueue
  - 创建了Native Looper对象
  - 调用epoll的epoll_create()/epoll_ctl()来完成对mWakeEventFd和mRequests的可读事件监听
- nativeDestroy()方法
  - 调用RefBase::decStrong()来减少对象的引用计数
  - 当引用计数为0时，则删除NativeMessageQueue对象
- nativePollOnce()方法
  - 调用Looper::pollOnce()来完成，空闲时停留在epoll_wait()方法，用于等待事件发生或者超时
- nativeWake()方法
  - 调用Looper::wake()来完成，向管道mWakeEventfd写入字符；

**总结：**真正的消息循环还是在java层的**Looper.loop()**方法，native层实现了对native消息的处理以及[阻塞](http://gityuan.com/2015/12/06/linux_epoll/)的机制。消息处理流程是先处理Native Message，再处理Native Request，最后处理Java Message。理解了该流程，也就明白有时上层消息很少，但响应时间却较长的真正原因。
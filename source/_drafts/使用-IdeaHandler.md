---
title: 使用 IdeaHandler
tags:
---



---


**知识链接：**

[你知道android的MessageQueue.IdleHandler吗？](https://wetest.qq.com/lab/view/352.html)


[Android 消息机制(Looper Handler MessageQueue Message)](https://mp.weixin.qq.com/s?__biz=MzIxNzU1Nzk3OQ==&mid=2247486513&idx=1&sn=892a944854aff3edb9f0b9fb85bc01c6&chksm=97f6b285a0813b9363b814b5d974a39990e23db91440bc8958eb5cf8758afb45e37cef6759c5&scene=38#wechat_redirect)解除了自己对 Message 如何插入到 MessageQueue 中的疑惑。


```
boolean enqueueMessage(Message msg, long when) {
      ......
    synchronized (this) {
       ......
        msg.when = when;
        Message p = mMessages;
        //检测当前头指针是否为空（队列为空）或者没有设置when 或者设置的when比头指针的when要前
        if (p == null || when == 0 || when < p.when) {
            //插入队列头部，并且唤醒线程处理msg
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
           // 几种情况要唤醒线程处理消息：1）队列是堵塞的 2)barrier，头部结点无target 3）当前msg是堵塞的
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // 将当前msg插入第一个比其when值大的结点前。
            prev.next = msg;
        }
        //调用Native方法进行底层操作，在这里把那个沉睡的主线程唤醒
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}

```
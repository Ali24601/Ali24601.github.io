---
layout: post
title: 物联网报文解析工具回顾
---

从入职华三到现在，一直在从事维护物联网报文解析工具的工作。这个工作总体比较简单，但是期间也有一些挑战，如使用多线程提升报文收发能力；实时显示心率图像的功能；统计数据中多标签页的支持。

# 工具总体结构如下图所示。

左侧AP、AC是运行在linux下的独立设备，右侧的Server为运行在Windows下的应用程序，采用MFC开发，二者通过交换机或者直连保持相互的畅通，且通过Socket进行Udp报文的通信。
![iotserver.jpg](https://i.loli.net/2018/09/04/5b8d5d310a3c9.jpg)

1. 左边的AP、AC在收到小报文之后经过打包、加密，形成加密的大包发送给Server。

2. Server点击"开始"之后，创建一个接收线程和4个处理线程处理报文数据。

3. 接收线程流程为：

    * 加载套接字库

    * 创建套接字

    * 绑定套接字（IP、port）

    * 调用recvfrom阻塞听

    * 接收到包之后sendto给AC、AP回应答（图中无体现），然后解密与解包，拆成一个个小报文，最后根据小报文中终端的ID做HASH，分配到0~3号队列中。

4. 0~3号处理线程从各自的队列中读取小报文，解析报文，然后交由UI显示；如果没有从队列读到报文，则Sleep一段时间。之所以取4个读写队列和4个处理线程是因为测试用的主机为2核。

5. 点击"停止"，Server不再接收并处理报文。

    * 置变量stopping为true，表示正在退出。

    * 等待4个处理线程退出（处理线程读到stopping为true，则退出循环体，线程自然退出）。

    * 关闭日志功能、关闭socket

    * 等待读取线程退出（写线程在socket关闭之后，退出阻塞读过程，返回Error为WSAEINTR）。

    * 清空4个读写队列。

## 涉及的通信方式

* 不同设备之间的socket通信

* 读线程、处理线程之间通过共享数据（4个队列）通信；点击"停止"按钮置stopping变量然后让处理线程退出的过程也是采用的共享数据的方式。

* 处理线程通过调用显示函数更新报表内容，这是MFC的消息队列实现的

* 多个处理线程在输出报表时进行一些临界区的操作（如同时写日志），需要加锁，这里采用的是互斥量mutex

## 为什么采用4个读写队列以及4个处理线程，而非1个读写队列和4个处理线程？

* 前一种方式由接收线程负责HASH，然后分配报文给处理线程，只需要保证读写队列在单生产者单消费者条件下是安全的即可；不好的一点是接收线程不仅负责了解密、拆包工作，还做了报文分配工作，需要知道报文格式中ID所在字段。

* 后一种方式需要有一个单生产者多消费者环境下安全的队列，在C++下比较难实现，而且由于读写过程中的竞争，容易出错，如果加读的锁，也会导致变慢。

## 为什么采用对终端ID做HASH的方式分配报文，而非整个报文内容做HASH？

选择终端ID做HASH分配报文是为了保证同一个终端的报文只存在于一个队列中，只能由同一个处理线程处理。这样，就不会发生多个线程同时更新某一个终端ID的报表内容的事情。后者会发生多个线程同时更新某一个终端ID的报表内容的事情，如果要解决的话需要加锁，相当于引入的不必要的锁。所以采用前一种方式，以线程职责不明确的代价，换来了效率的提升。

## 点击"停止"之后，界面线程等待处理线程、读取线程的自然退出；但是后者在退出之前，可能还需要更新界面，造成相互等待的情况，如何避免死锁。

调用`MsgWaitForMultipleObjects`边接收处理线程的消息（提示界面线程更新显示），边等待线程自然退出。收到处理线程的消息之后，依次调用`PeekMessage`、`TranslateMessage`、`DispatchMessage`把消息派发给指定对话框，完成界面更新。

{% highlight C++ %}
void CMyDlg::WaitExitThread(HANDLE *p_handle,unsigned int thread_count)
{
    DWORD dwRet = 0;
    DWORD count_left = thread_count;
    MSG msg;
    while(true)
    {
        dwRet = MsgWaitForMultipleObjects(count_left,p_handle,FALSE,INFINITE,QS_ALLINPUT);
        if(dwRet == WAIT_OBJECT_0+count_left)
        {
            //处理消息
            while(PeekMessage(&msg,NULL,0,0,PM_REMOVE))
            {
                TranslateMessage(&msg);
                DispatchMessage(&msg);
            }
        }
        else if (dwRet >=WAIT_OBJECT_0&&dwRet<WAIT_OBJECT_0+count_left)
        {
            //有线程退出
            int index = dwRet - WAIT_OBJECT_0;
            CloseHandle(p_handle[index]);
            if(index != count_left-1)
                p_handle[index]=p_handle[count_left-1];
            p_handle[count_left-1]=NULL;
            if(--count_left==0)
                break;
        }
        else
        {
            DWORD dErrCode = GetLastError();
            break;
        }
    }
}
{% endhighlight %}

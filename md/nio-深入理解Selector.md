# nio-深入理解Selector
## 概述
Selector提供了询问注册在其上的所有Channel是够已经准备好执行I/O操作的能力。
## 假如你设计，如何实现Channel托管？
1. 通过生产者/消费者的思想或者说是wait/notify的思路。首先需要集中管理Socket对象，分配一个或者多个线程去处理I/O操作，这些线程处于激活等待状态，当有新的远程客户端连接或者远程客户端的发送数据至服务端，这个时候激活一个线程或者加入某一个激活的线程去管理一个Socket连接或者唤醒托管该Socket连接的线程去执行I/O操作。性能好，设计较为复杂。
2. 通过轮询策略，首先所有的Socket被托管至一个或者多个线程，该线程会在一个合理的时间间隔去轮询这些Socket连接的流是否有数据代处理，有就去执行相关I/O操作。性能较差，实时性较差，实施简单。

### 上述Channel状态检查的问题
1. 每一次的状态检查都会至少执行一次系统调用。
2. 存在极端情况，每次系统调用后，就绪状态改变，在现在轮询开始之前是无法感知这个变化的，这就无法做到实时。
3. Channel检查就绪的方法read(ByteBuffer buffer)方法是，当就绪状态，很可能已经读取了数据，也可能没有读取，这就使得检查就绪和执行读取任务耦合在一起。

## BIO怎么实现Socket的托管呢？
* 首先accept,然后开辟一个线程去执行读任务，线程类执行read方法，该方法是阻塞的直到流中有数据才会唤醒。这里被阻塞线程当做了Socket监控器，JVM的线程调度充当了通知机制。在用户量大的时候，机器的内存和cpu占用率将急速上升，极可能导致服务器宕机。
* 可以通过将accept接收的Socket连接集中管理，通过分组的方式，通过一个或者几个少量的线程去执行任务，通过inputStrea的available方法，该方法是非阻塞的，去执行监控。有数据的话通过工作线程池去执行读写任务，或者入队，线程批量操作。这种方法较为合理，但是轮询检查的时间不太把控，很容易导致cpu负载过高或者状态响应不及时。

## 算了吧，这些想法都不太靠谱
真正就绪选择必须由系统负责，因为所有的流操作底层操作都是系统，只有系统知道流的状态，所以系统充当监视器将极大的提升了效率，也是最终的方案解决方案。

## Selector处理流程
![](https://raw.githubusercontent.com/yinzhongzheng/note/master/img/selectorflow.png?raw=true)
1. selector管理的channel都是非阻塞的。
2. 讲serverChanner和socketChannel注册到selector上,selector不是采用轮询的机制来监控channel，而是建立在系统内核提供的多路分离函数的基础之上的，注册在selector上的channel都在有I/O事件的时候，系统会唤醒阻塞的select线程，这时候selector的publicselectkey将会加入该channel所关联的key，该集合只增加不会被移除。
3. 唤醒的select线程将从就绪set中取出每一个key来遍历感兴趣的事件，由于该set不会移除key，用户需要根据业务需求移除相关的key。在多线程情况下，移除操作不是线程安全的，一个线程的remove操作将影响其他线程的iterator的结构，会抛出异常。
4. 只有ServerSocketChanner才有accept的权限，该事件是与客户端建立连接的事件，放回一个SocketChannel对象，将该连接通道再注册到selector中，大部分用来监听read事件，即socket流中有客户端的数据。
5. selector支持异步的移除key，Selector中cancelledKeys保存被移除的key，再下一次被系统唤醒的时候将会根据cancelledKeys更新publicKeys（注册的key），publicSelectedKeys（就绪状态的key，从一开始所有的就绪状态的key都在改set中，除非用户手动remove）。
6. 唤醒selector的方法有intercept、wakeup。非阻塞的select方法selectNow，直接返回结果。

## 个人理解
* selecto最大的点事借助了系统内核支持的select函数，这就省去了轮序检查的过程，大大提高了效率和降低检查开销。
* channel是selector管理的对象，channel是对socket的一种封装，底层实际操作读写的还是socket流，channel实现了读写的非阻塞，提高了系统的并发量，selector只能托管非阻塞的channel。非阻塞不代表是异步，不存在回调的概念。
* ByteBuffer是channel读写的最小单元,buffer又分为JVM内存管理的buffer和direct buffer（内核可以直接填充和排满）。通过direct buffer大大提高了数据集传输效率，也就是常说的0拷贝，不需要进行中转。

## I/O 多路复用技术
I/O多路复用是将多个I/O的阻塞复用到一个select的阻塞上。从而使系统在单线程的情况下可以处理多个客户端请求。
1. select/pool
进程将一个或多个fd（socket描述符）传递给select或pool调用，阻塞在select操作上，select/pool会检测多个fd是否处于就绪状态。select/pool是顺序扫描所有的fd，来判断状态的，支持fd的数量有限。缺点一个是扫描性能差，不能支持大量的fd的数量。
2. epool
epool模型不是通过扫描的方式，是通过基于事件驱动的方式代替低效的扫描。每个fd上面都有一个callback，当socke处于就绪状态，才会主动去调用callback函数，epool通过该callback来实现。

## 消息传递
将fd的消息从内核通知给用户空间，传统的做法是通过复制的方式，将内核数据拷贝到用户空间。epool通过内核和用户空间mmap同一块内存来避免数据的拷贝，来提高消息传递的效率。

## 异步I/O
通过系统调用告知内核开始某个操作，此事该线程将不会阻塞，内核在整个操作完成后（包括数组从内核复制到用户缓冲区），会以回调的方式通知调用者。而epool是告知调用者哪些fd已经就绪。

## 个人主页
https://github.com/yinzhongzheng
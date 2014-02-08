Channel与Reactor模式
--------

## 一：Netty、NIO、多线程？

时隔很久终于又更新了！之前一直迟迟未动也是因为积累不够，后面比较难下手。过年期间[@李林锋hw](http://weibo.com/lilinfeng)发布了一个Netty5.0架构剖析和源码解读 [http://vdisk.weibo.com/s/C9LV9iVqH13rW/1391437855](http://vdisk.weibo.com/s/C9LV9iVqH13rW/1391437855)，看完也是收获不少。前面的文章我们分析了Netty的结构，这次咱们来分析最错综复杂的一部分-Netty中的多线程以及NIO的应用。

理清NIO与Netty的关系之前，我们必须先要来看看Reactor模式。Netty是一个典型的多线程的Reactor模式的使用，理解了这部分，在宏观上理解Netty的NIO及多线程部分就不会有什么困难了。

本篇文章依然针对Netty 3.7，不过因为也看过一点Netty 5的源码，所以会有一点介绍。

## 二：Reactor，反应堆还是核电站？

Reactor是一种广泛应用在服务器端开发的设计模式。Reactor中文大多译为“反应堆”，我当初接触这个概念的时候，就感觉很厉害，是不是它的原理就跟“核反应”差不多？后来才知道其实没有什么关系，从Reactor的兄弟“Proactor”（多译为前摄器）就能看得出来，这两个词的中文翻译其实都不是太好，不够形象。实际上，Reactor模式又有别名“Dispatcher”或者“Notifier”，我觉得这两个都更加能表明它的本质。

那么，Reactor模式究竟是个什么东西呢？这要从事件驱动的开发方式说起。我们知道，对于应用服务器，一个主要规律就是，CPU的处理速度是要远远快于IO速度的，如果CPU为了IO操作（例如从Socket读取一段数据）而阻塞显然是不划算的。好一点的方法是分为多进程或者线程去进行处理，但是这样会带来一些进程切换的开销，试想一个进程一个数据读了500ms，期间进程切换到它3次，但是CPU却什么都不能干，就这么切换走了，是不是也不划算？

这时先驱们找到了事件驱动，或者叫回调的方式，来完成这件事情。这种方式就是，应用业务向一个中间人注册一个回调（event handler），当IO就绪后，就这个中间人产生一个事件，并通知此handler进行处理。*这种回调的方式，也赢到了“好莱坞原则”（Hollywood principle）-“Don't call us, we'll call you”，在我们熟悉的IoC中也有用到。看来软件开发真是互通的！*

好了，我们现在来看Reactor模式。在前面事件驱动的例子里有个问题：我们如何知道IO就绪这个事件，谁来充当这个中间人？Reactor模式的答案是：由一个不断等待和循环的单独进程（线程）来做这件事，它接受所有handler的注册，并负责先操作系统查询IO是否就绪，在就绪后就调用指定handler进行处理，这个角色的名字就叫做Reactor。

Java中的NIO可以很好的和Reactor模式结合。关于NIO中的Reactor模式，我想没有什么资料能比Doug Lea大神（不知道Doug Lea？看看JDK集合包和并发包的作者吧）在[《Scalable IO in Java》](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)解释的更简洁和全面了。NIO中Reactor的核心是`Selector`，这里我贴一个最简单的Reactor的循环（这种循环结构又叫做`EventLoop`）。

```java
	public void run() {
		try {
			while (!Thread.interrupted()) {
				selector.select();
				Set selected = selector.selectedKeys();
				Iterator it = selected.iterator();
				while (it.hasNext())
					dispatch((SelectionKey) (it.next()));
				selected.clear();
			}
		} catch (IOException ex) { /* ... */
		}
	}
```

前面提到了Proactor模式，这又是什么呢？简单来说，Reactor模式里，操作系统只负责通知IO就绪，具体的IO操作（例如读写）仍然是要在业务进程里阻塞的去做的，而Proactor模式则更进一步，由操作系统将IO操作执行好（例如读取，会将数据直接读到内存buffer中），而handler只负责处理自己的逻辑，真正做到了IO与程序处理异步执行。所以我们一般又说Reactor是同步IO，Proactor是异步IO。

关于阻塞和非阻塞、异步和非异步，以及UNIX底层的机制，大家可以看看这篇文章[IO - 同步，异步，阻塞，非阻塞 （亡羊补牢篇）](http://blog.csdn.net/historyasamirror/article/details/5778378)，以及陶辉（《深入理解nginx》的作者）《高性能网络编程》的系列。

## 三：由Reactor出发来理解Netty

Reactor可以认为是设计模式的一种。我认为设计模式有两个好处，写的时候，可以指导程序设计，别人读的时候，也按照既定的模式来分析代码。

Netty中的Reactor模式其实是一种多线程的Reactor模型。Doug Lea大神（不知道Doug Lea？回头看看JDK集合包和并发包的作者吧）在《Scalable IO in Java》中提到了这种模式，借用一下图：

![Multiple Reactors][1]

  [1]: http://static.oschina.net/uploads/space/2013/1125/130828_uKWD_190591.jpeg
  
## 合：由Reactor出发来理解Netty  

参考资料：

* Scalable IO in Java [http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)
* Netty5.0架构剖析和源码解读 [http://vdisk.weibo.com/s/C9LV9iVqH13rW/1391437855](http://vdisk.weibo.com/s/C9LV9iVqH13rW/1391437855)
* Reactor pattern [http://en.wikipedia.org/wiki/Reactor_pattern](http://en.wikipedia.org/wiki/Reactor_pattern)
* Reactor - An Object Behavioral Pattern for Demultiplexing and Dispatching Handles for Synchronous Events [http://www.cs.wustl.edu/~schmidt/PDF/reactor-siemens.pdf](http://www.cs.wustl.edu/~schmidt/PDF/reactor-siemens.pdf)
* 高性能网络编程6--reactor反应堆与定时器管理 [http://blog.csdn.net/russell_tao/article/details/17452997](http://blog.csdn.net/russell_tao/article/details/17452997)
* IO - 同步，异步，阻塞，非阻塞 （亡羊补牢篇）[http://blog.csdn.net/historyasamirror/article/details/5778378](http://blog.csdn.net/historyasamirror/article/details/5778378)
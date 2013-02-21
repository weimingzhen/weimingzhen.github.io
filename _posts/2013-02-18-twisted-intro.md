---
layout: post
title: Twisted Info Twisted入门教程
category: study
---

英文原文： [http://krondo.com/blog/?page_id=1327](http://krondo.com/blog/?page_id=1327)  

译文原文： [新浪杨晓伟博客](http://blog.sina.com.cn/s/blog_704b6af70100py9f.html)


##### 第一部分：Twisted理论基础 

>作者:dave@http://krondo.com/?p=1209译者:杨晓伟(采用意译)

**前言:**

最近有人在Twisted邮件列表中提出诸如”为任务紧急的人提供一份Twisted介绍”的的需求。值得提前透露的是，这个序列并不会如他们所愿.尤其是介绍Twisted框架和基于Python 的异步编程而言，可能短时间无法讲清楚。因此,如果你时间紧急，这恐怕不是你想找的资料。

我相信如果对异步编程模型一无所知，快速的介绍同样无法让你对其有所理解，至少你得稍微懂点基础知识吧。我已经用Twisted框架几年了，因此思考过我当初是怎么学习它(学得很慢)并发现学习它的最大难度并不在Twisted本身,而在于对其模型的理解，只有理解了这个模型，你才能更好去写和理解异步程序的代码。大部分Twisted的代码写得很清晰，其在线文档也非常棒（至少在开源软件这个层次上可以这么说）。但如果不理解这个模型，不管是读Twisted源码还是使用Twisted的代码更或者是相关文档，你都会感到非常的伤脑筋。
 
因此，我会用前面几个部分来介绍这个模型以让你掌握它的机制，稍后会介绍一下Twisted的特点。实际上，一开始，我们并不会使用Twisted，相反，会使用简单的Python来说明一个异步模型是如何工作的。我们在初次学习Twisted的时，会从你平常都不会直接使用的底层的实现讲起。Twisted是一个高度抽象的体系，因此在使用它时，你会体会到其多层次性。但当你去学习尤其是尝试着理解它是如何工作时，这种为抽像而带来的多层次性会给你带来极大的理解难度。所以，我们准备来个从内到外，从低层开始学习它。

**模型：**

为了更好的理解异步编程模型的特点，我们来回顾一下两个大家都熟悉的模型。在阐述过程中，我们假设一个包含三个相互独立任务的程序。在此，除了规定这些任务都要完成自己工作外，我们先不作具体的解释，后面我们会慢慢具体了解它们。请注意：在此我用“任务”这个词，这意味着它需要完成一些事情。

第一个模型是单线程的同步模型，如图1所示：

![图1](http://s7.sinaimg.cn/bmiddle/704b6af749e572a29d256&690)

这是最简单的编程方式。在一个时刻，只能有一个任务在执行，并且前一个任务结束后一个任务才能开始。如果任务都能按照事先规定好的顺序执行，最后一个任务的完成意味着前面所有的任务都已无任何差错地完成并输出其可用的结果—这是多么简单的逻辑。
下面我们来呈现第二个模型，如图2所示：

![图2](http://s10.sinaimg.cn/middle/704b6af749e572c35e599&690)  
图2 线程模型

在这个模型中，每个任务都在单独的线程中完成。这些线程都是由操作系统来管理，若在多处理机、多核处理机的系统中可能会相互独立的运行，若在单处理机上，则会交错运行。关键点在于，在线程模式中，具体哪个任务执行由操作系统来处理。但编程人员则只需简单地认为：它们的指令流是相互独立且可以并行执行。虽然，从图示看起来很简单，实际上多线程编程是很麻烦的，你想啊，任务之间的要通信就要是线程之间的通信。线程间的通信那不是一般的复杂。什么邮箱、通道、共享内存、、、 唉：（
一些程序用多处理机而不是多线程来实现并行运算。虽然具体的编程细节是不同的，但对于我们要研究的模型来说是一样的。

下面我们来介绍一下异步编程模型，如图3所示

![图3](http://s14.sinaimg.cn/middle/704b6af749e572c6bec6d&690)  
图3 异步模型

在这个模型中，任务是交错完成，值得注意的是：这是在单线程的控制下。这要比多线程模型简单多了，因为编程人员总可以认为只有一个任务在执行，而其它的在停止状态。虽然在单处理机系统中，线程也是像图3那样交替进行。但作为程序员在使用多线程时，仍然需要使用图2而不是图3的来思考问题，以防止程序在挪到多处理机的系统上无法正常运行（考虑到兼容性）。间单线程的异步程序不管是在单处理机还是在多处理机上都 能很好的运行。

在异步编程模型与多线程模型之间还有一个不同：在多线程程序中，对于停止某个线程启动另外一个线程，其决定权并不在程序员手里而在操作系统那里，因此，程序员在编写程序过程中必须要假设在任何时候一个线程都有可能被停止而启动另外一个线程。相反，在异步模型中，一个任务要想运行必须显式放弃当前运行的任务的控制权。这也是相比多线程模型来说，最简洁的地方。
值得注意的是：将异步编程模型与同步模型混合在同一个系统中是可以的。但在介绍中的绝大多数时候，我们只研究在单个线程中的异步编程模型。

**动机**

我们已经看到异步编程模型之所以比多线程模型简单在于其单令流与显式地放弃对任务的控制权而不是被操作系统随机地停止。但是异步模型要比同步模型复杂得多。程序员必须将任务组织成序列来交替的小步完成。因此，若其中一个任务用到另外一个任务的输出，则依赖的任务（即接收输出的任务）需要被设计成为要接收系列比特或分片而不是一下全部接收。由于没有实质上的并行，从我们的图中可以看出，一个异步程序会花费一个同步程序所需要的时间，可能会由于异步程序的性能问题而花费更长的时间。
 
因此，就要问了，为什么还要使用异步模型呢？ 在这儿，我们至少有两个原因。首先，如果有一到两个任务需要完成面向人的接口，如果交替执行这些任务，系统在保持对用户响应的同时在后台执行其它的任务。因此，虽然后台的任务可能不会运行的更快，但这样的系统可能会欢迎的多。
然而，有一种情况下，异步模型的性能会高于同步模型，有时甚至会非常突出，即在比较短的时间内完成所有的任务。这种情况就是任务被强行等待或阻塞，如图4所示：

![图4](http://s9.sinaimg.cn/middle/704b6af749e572cb02e98&690)  

图4 同步模型中出现阻塞


在图4中，灰色的部分代表这段时间某个任务被阻塞。为什么要阻塞一个任务呢？最直接的原因就是等待I/O的完成：传输数据或来自某个外部设备。一个典型的CPU处理数据的能力是硬盘或网络的几个数量级的倍数。因此，一个需要进行大I/O操作的同步程序需要花费大量的时间等待硬盘或网络将数据准备好。正是由于这个原因，同步程序也被称作为阻塞程序。

从图4中可以看出，一个可阻塞的程序，看起来与图3描述的异步程序有点像。这不是个巧合。异步程序背后的最主要的特点就在于，当出现一个任务像在同步程序一样出现阻塞时，会让其它可以执行的任务继续执行，而不会像同步程序中那样全部阻塞掉。因此一个异步程序只有在没有任务可执行时才会出现“阻塞”，这也是为什么异步程序被称为非阻塞程序的原因。
任务之间的切换要不是此任务完成，要不就是它被阻塞。由于大量任务可能会被阻塞，异步程序等待的时间少于同步程序而将这些时间用于其它实时工作的处理（如与人打交道的接口），这样一来，前者的性能必然要高很多。

与同步模型相比，异步模型的优势在如下情况下会得到发挥：

1. 有大量的任务，因此在一个时刻至少有一个任务要运行  
2. 任务执行大量的I/O操作，这样同步模型就会在因为任务阻塞而浪费大量的时间  
3. 任务之间相互独立，以至于任务内部的交互很少。  

这些条件大多在CS模式中的网络比较繁忙服务器端出现（如WEB服务器）。每个任务代表一个客户端进行接收请求并回复的I/O操作。客户的请求（相当于读操作）都是相互独立的。因此一个网络服务是异步模型的典型代表，这也是为什么twisted是第一个也是最棒的网络库。

##### 第二部分：异步编程初探与reactor模式

>作者:dave@http://krondo.com/?p=1247译者:杨晓伟(采用意译)

**第二部分:低效的诗歌服务器来启发对Twisted机制的理解**

这个系列是从这里开始的，欢迎你再次来到这里来。现在我们可能要写一些代码。在开始之前，我们都做出一些必要的假设。

**关于对你的假设**

在展开讨论前，我假设你已经有过用Python写同步程序的经历并且至少知道一点有关Python的Sockt编程的经验。如果你从没有写过Socket程序，或许你可以去看看Socket模块的文档，尤其是后面的示例代码。如果你没有用过Python的话，那后面的描述对你来说可能比看周易还痛苦。

**你所使用的计算机的情况**

我一般是在Linux上使用Twisted，这个系列的示例代码也是在Linux下完成的。首先声明的是我并没有故意让代码失去平台无关性，但我所讲述的一些内容确实可能仅仅适应于Linux和其它的类Unix（比如MAC OSX或FreeBSD）。WIndows是个奇怪诡异的地方（？？为什么这么评价Windows呢），如果你想尝试在它上面学习这个系列，抱歉，如果出了问题，我无法提供任何帮助。
并且假设你已经安装了近期版本的Python和Twisted。我所提供的示例示例代码是基于Python2.5和Twisted8.2.0。
你可以在单机上运行所有的示例代码，也可以在网络系统上运行它们。但是为了学习异步编程的机制，单机上学习是比较理想的。

**获取代码的方法**

使用git工具来获取Dave的最新示例代码。在shell或其它命令行上输入以下命令（假设已经安装git）：

    git clone git://github.com/jdavisp3/twisted-intro.git
    
下载结束后，解压并进入第一层文件夹（你可以看到有个README文件）。

**低效的诗歌服务器**

虽然CPU的处理速度远远快于网络，但网络的处理速度仍然比人脑快，至少比人类的眼睛快。因此，想通过网络来获得CPU的视角是很困难的，尤其是在单机的回环模式中数据流全速传输时，更是困难重重。
我们所需要的是一个慢速低效诗歌服务器，其用人为的可变延时来体现影响结果。毕竟服务器要提供点东西吗，我们就提供诗歌好了。目录下面有个子目录专门存放诗歌用的。
最简单的慢速诗歌服务器在blocking-server/slowpoetry.py中实现。你可用下面的方式来运行它。    

    python blocking-server/slowpoetry.py poetry/ecstasy.txt
    
上面这个命令将启动一个阻塞的服务器，其提供“Ecstasy”这首诗。现在我们来看看它的源码内容，正如你所见，这里面并没有使用任何Twisted的内容，只是最基本的Socket编程操作。它每次只发送一定字节数量的内容，而每次中间延时一段时间。默认的是每隔0.1秒发送10个比特，你可以通过-delay和
-num-bytes参数来设置。例如每隔5秒发送50比特：

    python blocking-server/slowpoetry.py --num-bytes 50 –delay 5poetry/ecstasy.txt
    
当服务器启动时，它会显示其所监听的端口号。默认情况下，端口号是在可用端口号池中随机选择的。你可能想使用固定的端口号，那么无需更改代码，只需要在启动命令中作下修改就OK了，如下所示：
    
    python blocking-server/slowpoetry.py --port 10000 poetry/ecstasy.txt

如果你装有netcat工具，可以用如下命令来测试你的服务器（也可以用telnet）：

    netcat localhost 10000    
    
如果你的服务器正常工作，那么你就可以看到诗歌在你的屏幕上慢慢的打印出来。对！你会注意到每次服务器都会发送过一行的内容过来。一旦诗歌传送完毕，服务器就会关闭这条连接。

默认情况下，服务器只会监听本地回环的端口。如果你想连接另外一台机子的服务器，你可以指定其IP地址内容，命令行参数是 -iface选项。
不仅是服务器在发送诗歌的速度慢，而且读代码可以发现，服务器在服务一个客户端时其它连接进来的客户端只能处于等待状态而得不到服务。这的确是一个低效慢速的服务器，要不是为了学习，估计没有任何其它用处。

**阻塞模式的客户端**

在示例代码中有一个可以从多个服务器中顺序（一个接一个）地下载诗歌的阻塞模式的客户端。下面让这个客户端执行三个任务，正如第一个部分图1描述的那样。首先我们启动三个服务器，提供三首不同的诗歌。在命令行中运行下面三条命令：

    python blocking-server/slowpoetry.py --port 10000 poetry/ecstasy.txt --num-bytes 30 
    python blocking-server/slowpoetry.py --port 10001 poetry/fascination.txt 
    python blocking-server/slowpoetry.py --port 10002 poetry/science.txt    
        
如果在你的系统中上面那些端口号有正在使用中，可以选择其它没有被使用的端口。注意，由于第一个服务器发送的诗歌是其它的三倍，这里我让第一个服务器使用每次发送30个字节而不是默认的10个字节，这样一来就以3倍于其它服务器的速度发送诗歌，因此它们会在几乎相同的时间内完成工作。

现在我们使用阻塞模式的客户端来获取诗歌，运行如下所示的命令：

    python blocking-client/get-poetry.py 10000 10001 10002

如果你修改了上面服务口器的端口，你需要在这里时行相应的修改以保持一致。由于这个客户端采用的是阻塞模式，因此它会一首一首的下载，即只有在完成一首时才会开始下载另外一首。这个客户端会像下面这样打印出提示信息而不是将诗歌打印出来：

    Task 1: get poetry from: 127.0.0.1:10000
    Task 1: got 3003 bytes of poetry from 127.0.0.1:10000 in 0:00:10.126361 
    Task 2: get poetry from: 127.0.0.1:10001 
    Task 2: got 623 bytes of poetry from 127.0.0.1:10001 in 0:00:06.321777
    Task 3: get poetry from: 127.0.0.1:10002 
    Task 3: got 653 bytes of poetry from 127.0.0.1:10002 in 0:00:06.617523
    Got 3 poems in 0:00:23.065661

这图1最典型的文字版了，每个任务下载一首诗歌。你运行后可能显示的时间会与上面有所差别，并且也会随着你改变服务器的发送时间参数而改变。尝试着更改一下参数来观测一下效果。

**异步模式的客户端**

现在，我们来看看不用Twisted构建的异步模式的客户端。首先，我们先运行它试试。启动使用前面的三个端口来启动三个服务器。如果前面开启的还没有关闭，那就继续用它们好了。接下来，我们通过下面这段命令来启动我们的异步模式的客户端：

    python async-client/get-poetry.py 10000 10001 10002 

你或许会得到类似于下面的输出：

    Task 1: got 30 bytes of poetry from 127.0.0.1:10000 
    Task 2: got 10 bytes of poetry from 127.0.0.1:10001
    Task 3: got 10 bytes of poetry from 127.0.0.1:10002
    Task 1: got 30 bytes of poetry from 127.0.0.1:10000 
    Task 2: got 10 bytes of poetry from 127.0.0.1:10001
    ...
    Task 1: 3003 bytes of poetry
    Task 2: 623 bytes of poetry
    Task 3: 653 bytes of poetry
    Got 3 poems in 0:00:10.133169

这次的输出可能会比较长，这是由于在异步模式的客户端中，每次接收到一段服务器发送来的数据都要打印一次提示信息，而服务器是将诗歌分成若干片段发送出去的。值得注意的是，这些任务相互交错执行，正如第一部分图3所示。

尝试着修改服务器的设置（如将一个服务器的延时设置的长一点），来观察一下异步模式的客户端是如何针对变慢的服务器自动调节自身的下载来与较快的服务器保持一致。这正是异步模式在起作用。

还需要值得注意的是，根据上面的设置，异步模式的客户端仅在10秒内完成工作，而同步模式的客户端却使用了23秒。现在回忆一下第一部分中图3与图4.通过减少阻塞时间，我们的异步模式的客户端可以在更短的时间里完成下载。诚然，我们的异步客户端也有些阻塞发生，那是由于服务器太慢了。由于异步模式的客户端可以在不同的服务器来回切换，它比同步模式的客户产生的阻塞就少得多。

**更近一步的观察**

现在让我们来读一下异步模式客户端的代码。注意其与同步模式客户端的差别：
 
1. 异步模式客户端一次性与全部服务器完成连接，而不像同步模式那样一次只连接一个。

2. 用来进行通信的Socket方法是非阻塞模的，这是通过调用setblocking(0)来实现的。

3. select模块中的select方法是用来识别是其监视的socket是否有完成数据接收的，如果没有它就处于阻塞状态。

4. 当从服务器中读取数据时，会尽量多地从Sockt读取数据直到它阻塞为止，然后读下一个Sockt接收的数据（如果有数据接收的话）。这意味着我们需要跟踪记录从不同服务器传送过来诗歌的接收情况（因为，一首诗的接收并不是连续完成，所以需要保证每个任务的可连续性，就得有冗余的信息来完成这一工作）。

异步模式中客户端的核心就是最高层的循环体，即get_poetry函数。这个函数可以被拆分成两个步骤：

1. 使用select函数等待所有Socket，直到至少有一个socket有数据到来。
2. 对每个有数据需要读取的socket，从中读取数据。但仅仅只是读取有效数据，不能为了等待还没来到的数据而发生阻塞。
3. 重复前两步，直到所有的socket被关闭。

可以看出，同步模式客户端也有个循环体（在main函数内），但是这个循环体的每个迭代都是完成一首诗的下载工作。而在异步模式客户端的每次迭代过程中，我们可以完成所有诗歌的下载或者是它们中的一些。我们并不知道在一个迭代过程中，在下载那首诗，或者一次迭代中我们下载了多少数据。这些都依赖于服务器的发送速度与网络环境。我们只需要select函数告诉我们那个socket有数据需要接收，然后在保证不阻塞程序的前提下从其读取尽量多的数据。

如果在服务器端口固定的条件下，同步模式的客户端并不需要循环体，只需要顺序罗列三个get_poetry 就可以了。但是我们的异步模式的客户端必须要有一个循环体来保证我们能够同时监视所有的socket端。这样我们就能在一次循环体中处理尽可能多的数据。
这个利用循环体来等待事件发生，然后处理发生的事件的模型非常常见，而被设计成为一个模式：reactor模式。其图形化表示如图5所示：

![图5](http://s11.sinaimg.cn/middle/704b6af749e5a3f41e86a&690)

这个循环就是个”reactor“（反应堆），因为它等待事件的发生然对其作为相应的反应。正因为如此，它也被称作事件循环。由于交互式系统都要进行I/O操作，因此这种循环也有时被称作select loop,这是由于select调用被用来等待I/O操作。因此，在本程序中的select循环中，一个事件的发生意味着一个socket端处有数据来到。值得注意的是，select并不是唯一的等待I/O操作的函数，它仅仅是一个比较古老的函数而已（因此才被用的如此广泛）。现在有一些新API可以完成select的工作而且性能更优，它们已经在不同的系统上实现了。不考虑性能上的因素，它们都完成同样的工作：监视一系列sockets（文件描述符）并阻塞程序，直到至少有一个准备好时行I/O操作。

严格意义上来说，我们的异步模式客户端中的循环并不是reactor模式，因为这个循环体并没有独立于业务处理（在此是接收具体个服务器传送来的诗歌）之外。它们被混合在一起。一个真正reactor模式的实现是需要实现循环独立抽象出来并具有如下的功能：

1. 监视一系列与你I/O操作相关的文件描述符（description)
2. 不停地向你汇报那些准备好I/O操作的文件描述符

一个设计优秀的reactor模式实现需要做到：

1. 处理所有不同系统会出现的I/O事件
2. 提供优雅的抽象来帮助你在使用reactor时少花些心思去考虑它的存在
3. 提供你可以在抽象层外（treactor实现）使用的公共协议实现。

好了，我们上面所说的其实就是Twisted—健壮、跨平台实现了reactor模式并含有很多附加功能。
在第三部分中，实现Twisted版的下载诗歌服务时，我们将开始写一些简单的Twisted程序。

##### 第三部分：初步认识Twisted

>作者:dave@http://krondo.com/?p=1333译者:杨晓伟(采用意译)

**用twisted的方式实现前面的内容**

最终我们将使用twisted的方式来重新实现我们前面的异步模式客户端。不过，首先我们先稍微写点简单的twisted程序来认识一下twisted。
最最简单的twisted程序就是下面的代码，其在twisted-intro目录中的basic-twisted/simple.py中。

    from twisted.internet import reactor
    reactor.run()

可以用下面的命令来运行它：
    
    python basic-twisted/simple.py

正如在第二部分所说的那样，twisted是实现了Reactor模式的，因此它必然会有一个对象来代表这个reactor或者说是事件循环，而这正是twisted的核心。上面代码的第一行引入了reactor，第二行开始启动事件循环。

这个程序什么事情也不做。除非你通过ctrl+c来终止它，否则它会一直运行下去。正常情况下，我们需要给出事件循环或者文件描述符来监视I/O（连接到某个服务器上，比如说我们那个诗歌服务器）。后面我们会来介绍这部分内容，现在这里的reactor被卡住了。值得注意的是，这里并不是一个在不停运行的简单循环。如果你在桌面上有个CPU性能查看器，可以发现这个循环体不会带来任何性能损失。实际上，这个reactor被卡住在第二部分图5的最顶端，等待永远不会到来的事件发生（更具体点说是一个调用select函数，却没有监视任何文件描述符)。

下面我们会让这个程序丰富起来，不过事先要说几个结论：

1. Twisted的reactor只有通过调用reactor.run()来启动。
2. reactor循环是在其开始的进程中运行，也就是运行在主进程中。
3. 一旦启动，就会一直运行下去。reactor就会在程序的控制下（或者具体在一个启动它的线程的控制下）。
4. reactor循环并不会消耗任何CPU的资源。
5. 并不需要显式的创建reactor，只需要引入就OK了。

最后一条需要解释清楚。在Twisted中，reactor是Singleton（也是一种模式），即在一个程序中只能有一个reactor，并且只要你引入它就相应地创建一个。上面引入的方式这是twisted默认使用的方法，当然了，twisted还有其它可以引入reactor的方法。例如，可以使用twisted.internet.pollreactor中的系统调用来poll 来代替select方法。

若使用其它的reactor，需要在引入twisted.internet.reactor前安装它。下面是安装pollreactor的方法：

    from twisted.internet import pollreactor
    pollreactor.install()

如果你没有安装其它特殊的reactor而引入了twisted.internet.reactor，那么Twisted会为你安装selectreactor。正因为如此，习惯性做法不要在最顶层的模块内引入reactor以避免安装默认reactor，而是在你要使用reactor的区域内安装。

下面是使用 pollreactor重写上上面的程序，可以在basic-twisted/simple-poll.py文件中找到中找到：

    from twited.internet import pollreactor
    pollreactor.install()
    from twisted.internet import reactor
    reactor.run()

上面这段代码同样没有做任何事情。
后面我们都会只使用默认的reactor，就单纯为了学习来说 ，所有的不同的reactor做的事情都一样。

**你好，Twisted**

我们得用Twisted来做什么吧。下面这段代码在reactor循环开始后向终端打印一条消息：

    def hello():
        print 'Hello from the reactor loop!'
        print 'Lately I feel like I\'m stuck in a rut.'
        
    from twisted.internet import reactor 
    reactor.callWhenRunning(hello)
    print 'Starting the reactor.'
    reactor.run()

这段代码可以在basic-twisted/hello.py中找到。运行它，会得到如下结果：

    Starting the reactor. 
    Hello from the reactor loop!
    Lately I feel like I'm stuck in a rut.

仍然需要你手动来关掉程序，因为它在打印完毕后就又卡住了。

值得注意的是，hello函数是在reactor启动后被调用的。这意味是reactor调用的它，也就是说Twisted在调用我们的函数。我们通过调用reactor的callWhenRunning函数，并传给它一个我们想调用函数的引用来实现hello函数的调用。当然，我们必须在启动reactor之前完成这些工作。
我们使用回调来描述hello函数的引用。回调实际上就是交给Twisted（或者其它框架）的一个函数引用，这样Twisted会在合适的时间调用这个函数引用指向的函数，具体到这个程序中，是在reactor启动的时候调用。由于Twisted循环是独立于我们的代码，我们的业务代码与reactor核心代码的绝大多数交互都是通过使用Twisted的APIs回调我们的业务函数来实现的。

我们可以通过下面这段代码来观察Twisted是如何调用我们代码的：

    import traceback

    def stack():
        print 'The python stack:'
        traceback.print_stack()

    from twisted.internet import reactor
    
    reactor.callWhenRunning(stack)
    reactor.run()

这段代码的文件是 basic-twisted/stack.py。不出意外，它的输出是：

    The python stack: 
    ... reactor.run() <-- This is where we called the reactor 
    ... ... <-- A bunch of Twisted function calls ... 
    traceback.print_stack() <-- The second line in the stack function

不用考虑这其中的若干Twisted本身的函数。只需要关心reactor.run()与我们自己的函数调用之间的关系即可。

有关回调的一些其它说明：

Twisted并不是唯一使用回调的框架。许多历史悠久的框架都已在使用它。诸多GUI的框架也是基于回调来实现的，如GTK和QT。
交互式程序的编程人员特别喜欢回调。也许喜欢到想嫁给它。也许已经这样做了。但下面这几点值得我们仔细考虑下：

1. reactor模式是单线程的。
2. 像Twisted这种交互式模型已经实现了reactor循环，意味无需我们亲自去实现它。
3. 我们仍然需要框架来调用我们自己的代码来完成业务逻辑。
4. 因为在单线程中运行，要想跑我们自己的代码，必须在reactor循环中调用它们。
5. reactor事先并不知道调用我们代码的哪个函数

这样的话，回调并不仅仅是一个可选项，而是游戏规则的一部分。


图6说明了回调过程中发生的一切：

![pic6](http://s4.sinaimg.cn/middle/704b6af749e6fd666db03&690)  
图6 reactor启用回调

图6揭示了回调中的几个重要特性：

1. 我们的代码与Twisted代码运行在同一个进程中。
2. 当我们的代码运行时，Twisted代码是处于暂停状态的。
3. 同样，当Twisted代码处于运行状态时，我们的代码处于暂停状态。
4. reactor事件循环会在我们的回调函数返回后恢复运行。

在一个回调函数执行过程中，实际上Twisted的循环是被有效地阻塞在我们的代码上的。因此，因此我们应该确保回调函数不要浪费时间（尽快返回）。特别需要强调的是，我们应该尽量避免在回调函数中使用会阻塞I/O的函数。否则，我们将失去所有使用reactor所带来的优势。Twisted是不会采取特殊的预防措施来防止我们使用可阻塞的代码的，这需要我们自己来确保上面的情况不会发生。正如我们实际所看到的一样，对于普通网络I/O的例子，由于我们让Twisted替我们完成了异步通信，因此我们无需担心上面的事情发生。

其它也可能会产生阻塞的操作是读或写一个非socket文件描述符（如管道）或者是等待一个子进程完成。
如何从阻塞转换到非阻塞操作取决你具体的操作是什么，但是也有一些Twisted APIs会帮助你实现转换。值得注意的是，很多标准的Python方法没有办法转换为非阻塞方式。例如，os.system中的很多方法会在子进程完成前一直处于阻塞状态。这也就是它工作的方式。所以当你使用Twisted时，避开使用os.system。

**退出Twisted**

原来我们可以使用reactor的stop方法来停止Twisted的reactor。但是一旦reactor停止就无法再启动了。（Dave的意思是，停止就退出程序了），因此只有在你想退出程序时才执行这个操作。

下面是退出代码，代码文件是basic-twisted/countdown.py：

    class Countdown(object):
        counter = 5

        def count(self):
            from twisted.internet import reactor
            if self.counter == 0:
                reactor.stop()
            else:
                print self.counter, '...'
                self.counter -= 1

            reactor.callLater(1, self.count)

        from twisted.internet import reactor
        
        reactor.callWhenRunning(Countdown().count)
         
        print 'Start!'
        reactor.run()
        print 'Stop!'

在这个程序中使用了callLater函数为Twisted注册了一个回调函数。callLater中的第二个参数是回调函数，第一个则是说明你希望在将来几秒钟时执行你的回调函数。那Twisted如何来在指定的时间执行我们安排好的的回调函数。由于程序并没有监听任何文件描述符，为什么它没有像前那些程序那样卡在select循环上？select函数，或者其它类似的函数，同样会接纳一个超时参数。如果在只提供一个超时参数值并且没有可供I/O操作的文件描述符而超时时间到时，select函数同样会返回。因此，如果设置一个0的超时参数，那么会无任何阻塞地立即检查所有的文件描述符集。

你可以将超时作为图5中循环等待中的一种事件来看待。并且Twisted使用超时事件来确保那些通过callLater函数注册的延时回调在指定的时间执行。或者更确切的说，在指定时间的前后会执行。如果一个回调函数执行时间过长，那么下面的延时回调函数可能会被相应的后延执行。Twisted的callLater机制并不为硬实时系统提供任何时间上的保证。

下面是上面程序的输出：

    Start! 
    5 ... 
    4 ...
    3 ... 
    2 ...
    1 ... 
    Stop!

**捕获它，Twisted**

由于Twisted经常会在回调中结束调用我们的代码，因此你可能会想，如果我们的回调函数中出现异常会发生什么状况。（Dave的意思是说，在结束我们的回调函数后会再次回到Twisted代码中，若在我们的回调中发生异常，那是不是异常会跑到Twisted代码中，而造成不可想象的后果 ）让我们来试试，在basic-twisted/exception.py中的程序会在一个回调函数中引发一个异常，但是这不会影响下一个回调：

    def falldown():
        raise Exception('I fall down.')
     
    def upagain():
        print 'But I get up again.'
        reactor.stop()
     

    from twisted.internet import reactor
     
    reactor.callWhenRunning(falldown)
    reactor.callWhenRunning(upagain)
     
    print 'Starting the reactor.'
    reactor.run()

当你在命令行中运时，会有如下的输出：

    Starting the reactor. Traceback (most recent call last):
    ... # I removed most of the traceback
    exceptions.Exception: I fall down.
    But I get up again.

注意，尽管我们看到了因第一个回调函数引发异常而出现的跟踪栈，第二个回调函数依然能够执行。如果你将reactor.stop()注释掉的话，程序会继续运行下去。所以说，reactor并不会因为回调函数中出现失败（虽然它会报告异常）而停止运行。

网络服务器通常需要这种健壮的软件。它们通常不希望由于一个随机的Bug导致崩溃。也并不是说当我们发现自己的程序内部有问题时，就垂头丧气。只是想说Twisted能够很好的从失败的回调中返回并继续执行。

**请继续讲解诗歌服务器**

现在，我们已经准备好利用Twisted来搭建我们的诗歌服务器。在第4部分，我们会实现我们的异步模式的诗歌服务器的Twisted版。


#####第四部分：由Twisted支持的诗歌客户端

**第一个twisted支持的诗歌服务器**

尽管Twisted大多数情况下用来写服务器代码，为了一开始尽量从简单处着手，我们首先从简单的客户端讲起。
让我们来试试使用Twisted的客户端。源码在twisted-client-1/get-poetry.py。首先像前面一样要开启三个服务器：

    python blocking-server/slowpoetry.py --port 10000 poetry/ecstasy.txt --num-bytes 30
    python blocking-server/slowpoetry.py --port 10001 poetry/fascination.txt
    python blocking-server/slowpoetry.py --port 10002 poetry/science.txt

并且运行客户端：

    python twisted-client-1/get-poetry.py 10000 10001 10002

你会看到在客户端的命令行打印出：

    Task 1: got 60 bytes of poetry from 127.0.0.1:10000
    Task 2: got 10 bytes of poetry from 127.0.0.1:10001
    Task 3: got 10 bytes of poetry from 127.0.0.1:10002
    Task 1: got 30 bytes of poetry from 127.0.0.1:10000 
    Task 3: got 10 bytes of poetry from 127.0.0.1:10002
    Task 2: got 10 bytes of poetry from 127.0.0.1:10001 
    ... 
    Task 1: 3003 bytes of poetry
    Task 2: 623 bytes of poetry
    Task 3: 653 bytes of poetry
    Got 3 poems in 0:00:10.134220

和我们的没有使用Twisted的非阻塞模式客户端打印的内容接近。这并不奇怪，因为它们的工作方式是一样的。

下面，我们来仔细研究一下它的源代码。

注意：正如我在第一部分说到，我们开始学习使用Twisted时会使用一些低层Twisted的APIs。这样做是为揭去Twisted的抽象层，这样我们就可以从内向外的来学习Tiwsted。但是这就意味着，我们在学习中所使用的APIs在实际应用中可能都不会见到。记住这么一点就行：前面这些代码只是用作练习，而不是写真实软件的例子。

可以看到，首先创建了一组PoetrySocket的实例。在PoetrySocket初始化时，其创建了一个网络socket作为自己的属性字段来连接服务器，并且选择了非阻塞模式：

    self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    self.sock.connect(address)
    self.sock.setblocking(0)

最终我们虽然会提高到不使用socket的抽象层次上，但这里我们仍然需要使用它。在创建完socket后，PoetrySocket通过方法addReader将自己传递给 reactor：

    # tell the Twisted reactor to monitor this socket for reading
    from twisted.internet import reactor
    reactor.addReader(self）

这个方法给Twisted提供了一个文件描述符来监视要发送来的数据。为什么我们不传递给Twisted一个文件描述符或回调函数而是一个对象实例？并且Twisted内部没有任何与这个诗歌服务相关的代码，它怎么知道该如何与我们的对象实例交互？相信我，我已经查看过了，打开twisted.internet.interfaces模块，和我一起来搞清楚是怎么回事。

**Twisted接口**

在twisted内部有很多被称作接口的子模块。每个都定义了一组接口类。由于在8.0版本中，Twisted使用zope.interface作为这些类的基类。但我们这里并不来讨论它其中的细节。我们只关心其在Twisted的子类，就是你看到的那些。

使用接口的核心目的之一就是文档化。作为一个python程序员，你肯定知道Duck Typing。（说实话我还真不懂这种编程，但通过查看资料，其实就是动态编程的思想，根据你的动作来确定你的类型）

翻阅twisted.internet.interfaces找到方法的addReader定义，它的定义在IReactorFDSet 中可以找到：

    def addReader(reader):
        """
        I add reader to the set of file descriptors to get read events for.
        @param reader: An L{IReadDescriptor} provider that will be checked for
                       read events until it is removed from the reactor with
                       L{removeReader}.
        @return: C{None}.
        """
IReactorFDSet是一个Twisted的reactor实现的接口。因此任何一个Twisted的reactor都会一个 addReader的方法，如同上面描述的一样工作。这个方法声明之所以没有self参数是因为它仅仅关心一个公共接口定义，self参数仅仅是接口实现时的一部分（在调用它时，也没有显式地传入一个self参数）。接口类永远不会被实例化或作为基类来继承实现。

注意1：技术上讲，IReactorFDSet只会由reactor实现用来监听文件描述符。具我所知，现在所有已实现reactor都会实现这个接口。

注意2：使用接口并不仅仅是为了文档化。zope.interface允许你显式地来声明一个类实现一个或多个接口，并提供运行时检查这些实现的机制。同样也提供代理这一机制，它可以动态地为一个没有实现某接口的类直接提供该接口。但我们这里就不做深入学习了。

注意3：你可能已经注意到接口与最近添加到Python中虚基类的相似性了。这里我们并不去分析它们之间的相似性与差异。若你有兴趣，可以读读Ptyhon项目的创始人Glyph写的一篇关于这个话题的文章。

根据文档的描述可以看出，addReader的reader参数是要实现IreadDescriptor接口的。这也就意味我们的PoetrySocket也必须这样做。

阅读接口模块我们可以看到下面这段代码：

    class IReadDescriptor(IFileDescriptor):
        def doRead():
            """
            Some data is available for reading on your descriptor.
            """
        
同时你会看到在我们的PoetrySocket类中有一个doRead方法。当其被Twisted的reactor调用时，就会采用异步的方式从socket中读取数据。因此，doRead其实就是一个回调函数，只是没有直接将其传递给reactor，而是传递一个实现此方法的对象实例。这也是Twisted框架中的惯例—不是直接传递实现某个接口的函数而是传递实现它的对象。这样我们通过一个参数就可以传递一组相关的回调函数。而且也可以让回调函数之间通过存储在对象中的数据进行通信。

那在PoetrySocket中实现其它的回调函数呢？注意到IReadDescriptor是IFileDescriptor的一个子类。这也就意味任何一个实现IReadDescriptor都必须实现IFileDescriptor。若是你仔细阅读代码会看到下面的内容：

    class IFileDescriptor(ILoggingContext):
        """
        A file descriptor.
        """
        def fileno():
            ...
        def connectionLost(reason):
        …
        
我将文档描述省略掉了，但这些函数的功能从字面上就可以理解：fileno返回我们想监听的文件描述符，connectionLost是当连接关闭时被调用。你也看到了，PoetrySocket实现了这些方法。

最后，IFileDescriptor继承了ILooggingContext，这里我不想再展现其源码。我想说的是，这就是为什么我们要实现一个logPrefix回调函数。你可以在interface模块中找到答案。

注意：你也许注意到了，当连接关闭时，在doRead中返回了一个特殊的值。我是如何知道的？说实话，没有它程序是无法正常工作的。我是在分析Twisted源码中发现其它相应的方法采取相同的方法。你也许想好好研究一下：但有时一些文档或书的解释是错误的或不完整的。因此可能当你搞清楚怎么回事时，我们已经完成第五部分了呵呵。

**更多关于回调的知识**

我们使用Twisted的异步客户端和前面的没有使用Twisted的异步客户非常的相似。两者都要连接它们自己的socket，并以异步的方式从中读取数据。最大的区别在于：使用Twisted的客户端并没有使用自己的select循环-而使用了Twisted的reactor。

doRead回调函数是非常重要的一个回调。Twisted调用它来告诉我们已经有数据在socket接收完毕。我可以通过图7来形象地说明这一过程：
 
![pic 7](http://simg.sinajs.cn/blog7style/images/common/sg_trans.gif)
 
图7 doRead回调过程

每当回调被激活，就轮到我们的代码将所有能够读的数据读回来然后非阻塞式的停止。正如我们第三部分说的那样，Twisted是不会因为什么异常状况（如没有必要的阻塞）而终止我们的代码。那么我们就故意写个会产生异常状况的客户端看看到底能发生什么事情。可以在twisted-client-1/get-poetry-broken.py中看到源代码。这个客户端与你前面看到的同样有两个异常状况出现：

1. 这个客户端并不没有选择非阻塞式的socket
2. doRead回调方法在socket关闭连接前一直在不停地读socket

现在让我们运行一下这个客户端：

    python twisted-client-1/get-poetry-broken.py 10000 10001 10002

我们出得到如同下面一样的输出：

    Task 1: got 3003 bytes of poetry from 127.0.0.1:10000
    Task 3: got 653 bytes of poetry from 127.0.0.1:10002 
    Task 2: got 623 bytes of poetry from 127.0.0.1:10001
    Task 1: 3003 bytes of poetry 
    Task 2: 623 bytes of poetry
    Task 3: 653 bytes of poetry
    Got 3 poems in 0:00:10.132753

可能除了任务的完成顺序不太一致外，和我先阻塞式客户端是一样的。这是因为这个客户端是一个阻塞式的。

由于使用了阻塞式的连接，就将我们的非阻塞式客户端变成了阻塞式的客户端。这样一来，我们尽管遭受了使用select的复杂但却没有享受到其带来的异步优势。

像诸如Twisted这样的事件循环所提供的多任务的能力是需要用户的合作来实现的。Twisted会告诉我们什么时候读或写一个文件描述符，但我们必须要尽可能高效而没有阻塞地完成读写工作。同样我们应该禁止使用其它各类的阻塞函数，如os.system中的函数。除此之外，当我们遇到计算型的任务（长时间占用CPU），最好是将任务切成若干个部分执行以让I/O操作尽可能地执行。

你也许已经注意到这个客户端所花费的时间少于先前那个阻塞的客户端。这是由于这个在一开始就与所有的服务建立连接，由于服务是一旦连接建立就立即发送数据，而且我们的操作系统会缓存一部分发送过来但尚读不到的数据到缓冲区中（缓冲区大小是有上限的）。因此就明白了为什么前面那个会慢了：它是在完成一个后再建立下一个连接并接收数据。

但这种小优势仅仅在小数据量的情况下才会得以体现。如果我们下载三首20M个单词的诗，那时OS的缓冲区会在瞬间填满，这样一来我们这个客户端与前面那个阻塞式客户端相比就没有什么优势可言了。

**结束语**

我没有过多地解释此部分第一个客户端的内容。你可能注意到了，connectionLost函数会在没有PoetrySocket等待诗歌后关闭reactor。由于我们的程序除了下载诗歌不提供其它服务，所以才会这样做。但它揭示了两个低层reactor的APIs：removeReader和getReaders。

还有与我们客户端使用的Readers的APIs类同的Writers的APIs，它们采用相同的方式来监视我们要发送数据的文件描述符。可以通过阅读interfaces文件来获取更多的细节。读和写有各自的APIs是因为select函数需要分开这两种事件（读或写可以进行的文件描述符）。当然了，可以等待即能读也能写的文件描述符。

第五部分，我们将使用Twisted的高层抽象方式实现另外一个客户端，并且学习更多的Twisted的接口与APIs。

#####第五部分：由Twited支持的诗歌下载服务客户端

**抽象地构建客户端**

在第四部分中，我们构建了第一个使用Twisted的客户端。它确实能很好地工作，但仍有提高的空间。

首先是，这个客户端竟然有创建网络端口并接收端口处的数据这样枯燥的代码。Twisted理应为我们实现这些例程性功能，省得我们每次写一个新的程序时都要去自己实现。Twisted这样做也将我们从像异步I/O操作中包括许多像异常处理这样的细节处理解放出来。更多的细节处理存在于多平台上运行我们的代码中。如果你那个下午有空，可以翻翻Twisted的WIN32实现源代码，看看里面有多少小针线是来处理跨平台的。

另一问题是与错误处理有关。当运行版本1的Twisted客户端来从并没有提供服务的端口上下载诗歌时，它就会崩溃。我们是可以修正这个错误，但通过下面我们要介绍Twisted的APIs来处理这些类型的错误会更简单。

最后，那个客户端也不能复用。如果有另一个模块需要通过我们的客户端下载诗歌呢？人家怎么知道你的诗歌已经下载完毕？我们不能用一个方法简单地将一首诗下载完成后再传给人家，而在之前让人家处于等待状态。这确实是一个问题，但我们不准备在这个部分解决这个问题—在未来的部分中一定会解决这个问题。

我们将会使用一些高层次的APIs和接口来解决第一、二个问题。Twisted框架是由众多抽象层松散地组合起来的。因此，学习Twisted也就意味着需要学习这些层都提供什么功能，例如每层都有哪些APIs，接口和实例可供使用。接下来我们会通过剖析Twisted最最重要的部分来更好地感受一下Twisted都是怎么组织的。一旦你对Twisted的整个结构熟悉了，学习新的部分会简单多了。

一般来说，每个Twisted的抽象都只与一个特定的概念相关。例如，第四部分中的客户端使用的IReadDescriptor，它就是“一个可以读取字节的文件描述符”的抽象。一个抽象往往会通过定义接口来指定那些想实现个抽象（也就是实现这个接口）对象的形为。在学习新的Twisted抽象概念时，最需要谨记的就是：

>多数高层次抽象都是在低层次抽象的基础上建立的，很少有另立门户的。

因此，你在学习新的Twisted抽象概念时，始终要记住它做什么和不做什么。特别是，如果一个早期的抽象A实现了F特性，那么F特性不太可能再由其它任何抽象来实现。另外，如果另外一个抽象需要F特性，那么它会使用A而不是自己再去实现F。（通常的做法，B可能会通过继承A或获得一个指向A实例的引用）

网络非常的复杂，因此Twisted包含很多抽象的概念。通过从低层的抽象讲起，我们希望能更清楚起看到在一个Twisted程序中各个部分是怎么组织起来的。

**核心的循环体**

第一个我们要学习的抽象，也是Twisted中最重要的，就是reactor。在每个通过Twisted搭建起来的程序中心处，不管你这个程序有多少层，总会有一个reactor循环在不停止地驱动程序的运行。再也没有比reactor提供更加基础的支持了。实际上，Twisted的其它部分（即除了reactor循环体）可以这样理解：它们都是来辅助X来更好地使用reactor，这里的X可以是提供Web网页、处理一个数据库查询请求或其它更加具体内容。尽管坚持像上一个客户端一样使用低层APIs是可能的，但如果我们执意那样做，那么我们必需自己来实现非常多的内容。而在更高的层次上，意味着我们可以少写很多代码。

但是当在外层思考与处理问题叶。很容易就忘记了reactor的存在了。在任何一个常见大小的Twisted程序中 ，确实很少会有直接与reactor的APIs交互。低层的抽象也是一样（即我们很少会直接与其交互）。我们在上一个客户端中用到的文件描述符抽象，就被更高层的抽象更好的归纳而至于我们很少会在真正的Twisted程序中遇到。（他们在内部依然在被使用，只是我们看不到而已）

至于文件描述符抽象的消息，这并不是一个问题。让Twisted掌舵异步I/O处理，这样我们就可以更加关注我们实际要解决的问题。但对于reactor不一样，它永远都不会消失。当你选择使用Twisted，也就意味着你选择使用Reactor模式，并且意味着你需要使用回调与多任务合作的“交互式”编程方式。如果你想正确地使用Twisted，你必须牢记reactor的存在。我们将在第六部分更加详细的讲解部分内容。但是现在要强调的是：

>图5与图6是这个系列中最最重要的图

我们还将用图来描述新的概念，但这两个图是需要你牢记在脑海中的。可以这样说，我在写Twisted程序时一直想着这两张图。

在我们付诸于代码前，有三个新的概念需要阐述清楚：Transports,Protocols,Protocol Factoies

**Transports**

Transports抽象是通过Twisted中interfaces模块中ITransport接口定义的。一个Twisted的Transport代表一个可以收发字节的单条连接。对于我们的诗歌下载客户端而言，就是对一条TCP连接的抽象。但是Twisted也支持诸如Unix中管道和UDP。Transport抽象可以代表任何这样的连接并为其代表的连接处理具体的异步I/O操作细节。
如果你浏览一下ITransport中的方法，可能找不到任何接收数据的方法。这是因为Transports总是在低层完成从连接中异步读取数据的许多细节工作，然后通过回调将数据发给我们。相似的原理，Transport对象的写相关的方法为避免阻塞也不会选择立即写我们要发送的数据。告诉一个Transport要发送数据，只是意味着：尽快将这些数据发送出去，别产生阻塞就行。当然，数据会按照我们提交的顺序发送。

通常我们不会自己实现一个Transport。我们会去实现Twisted提供的类，即在传递给reactor时会为我们创建一个对象实例。

**Protocols**

Twisted的Protocols抽象由interfaces模块中的IProtocol定义。也许你已经想到，Protocol对象实现协议内容。也就是说，一个具体的Twisted的Protocol的实现应该对应一个具体网络协议的实现，像FTP、IMAP或其它我们自己规定的协议。我们的诗歌下载协议，正如它表现的那样，就是在连接建立后将所有的诗歌内容全部发送出去并且在发送完毕后关闭连接。

严格意义上讲，每一个Twisted的Protocols类实例都为一个具体的连接提供协议解析。因此我们的程序每建立一条连接（对于服务方就是每接受一条连接），都需要一个协议实例。这就意味着，Protocol实例是存储协议状态与间断性（由于我们是通过异步I/O方式以任意大小来接收数据的）接收并累积数据的地方。

因此，Protocol实例如何得知它为哪条连接服务呢？如果你阅读IProtocol定义会发现一个makeConnection函数。这是一个回调函数，Twisted会在调用它时传递给其一个也是仅有的一个参数，即就是Transport实例。这个Transport实例就代表Protocol将要使用的连接。

Twisted包含很多内置可以实现很多通用协议的Protocol。你可以在twisted.protocols.basic中找到一些稍微简单点的。在你尝试写新Protocol时，最好是看看Twisted源码是不是已经有现成的存在。如果没有，那实现一个自己的协议是非常好的，正如我们为诗歌下载客户端做的那样。

**Protocol Factories**

因此每个连接需要一个自己的Portocol，而且这个Protocol是我们自己定义类的实例。由于我们会将创建连接的工作交给Twisted来完成，Twisted需要一种方式来为一个新的连接制定一个合适的协议。制定协议就是Protocol Factories的 工作了。

也许你已经猜到了，Protocol Factory的API由IProtocolFactory来定义，同样在interfaces模块中。Protocol Factory就是Factory模式的一个具体实现。buildProtocol方法在每次被调用时返回一个新Protocol实例。它就是Twisted用来为新连接创建新Protocol实例的方法。

**诗歌下载客户端2.0：第一滴心血**

好吧，让我们来看看由Twisted支持的诗歌下载客户端2.0。源码可以在这里twisted-client-2/get-poetry.py。你可以像前面一样运行它，并得到相同的输出。这也是最后一个在接收到数据时打印其任务的客户端版本了。到现在为止，对于所有Twisted程序都是交替执行任务并处理相对较少数量数据的，应该很清晰了。我们依然通过print函数来展示在关键时刻在进行什么内容，但将来客户端不会在这样繁锁。

在第二个版本中，sockets不会再出现了。我们甚至不需要引入socket模块也不用引用socket对象和文件描述符。取而代之的是，我们告诉reactor来创建到诗歌服务器的连接，代码如下面所示：

    factory = PoetryClientFactory(len(addresses))
     
    from twisted.internet import reactor
     
    for address in addresses:
        host, port = address
        reactor.connectTCP(host, port, factory)

我们需要关注的是connectTCP这个函数。前两个参数的含义很明显，不解释了。第三个参数是我们自定义的PoetryClientFactory类的实例对象。这是一个专门针对诗歌下载客户端的Protocol Factory，将它传递给reactor可以让Twisted为我们创建一个PeotryProtocol实例。

值得注意的是，从一开始我们既没有实现Factory也没有去实现Protocol，不像在前面那个客户端中我们去实例化我们PoetrySocket类。我们只是继承了Twisted在twisted.internet.protocol 中提供的基类。Factory的基类是twisted.internet.protocol.Factory，但我们使用客户端专用（即不像服务器端那样监听一个连接，而是主动创建一个连接）的ClientFactory子类来继承。

我们同样利用了Twisted的Factory已经实现了buildProtocol方法这一优势来为我们所用。我们要在子类中调用基类中的实现：

    def buildProtocol(self, address):
        proto = ClientFactory.buildProtocol(self, address)
        proto.task_num = self.task_num
        self.task_num += 1
        return proto

基类怎么会知道我们要创建什么样的Protocol呢？注意，我们的PoetryClientFactory中有一个protocol类变量：

    class PoetryClientFactory(ClientFactory):
     
        task_num = 1
     
        protocol = PoetryProtocol # tell base class what proto to build

基类Factory的实现buildProtocol过程是：安装（创建一个实例）我们设置在protocol变量上的Protocol类与在这个实例（此处即PoetryProtocol的实例）的factory属性上设置一个产生它的Factory的引用（此处即实例化PoetryProtocol的PoetryClientFactory）。这个过程如图8所示：

![pic 8](http://s8.sinaimg.cn/middle/704b6af749e993c8ccba7&690)

图8：Protocol的生成过程

正如我们提到的那样，位于Protocol对象内的factory属性字段允许在都由同一个factory产生的Protocol之间共享数据。由于Factories都是由用户代码来创建的（即在用户的控制中），因此这个属性也可以实现Protocol对象将数据传递回一开始初始化请求的代码中来，这将在第六部分看到。
值得注意的是，虽然在Protocol中有一个属性指向生成其的Protocol Factory，在Factory中也有一个变量指向一个Protocol类，但通常来说，一个Factory可以生成多个Protocol。

在Protocol创立的第二步便是通过makeConnection与一个Transport联系起来。我们无需自己来实现这个函数而使用Twisted提供的默认实现。默认情况是，makeConnection将Transport的一个引用赋给（Protocol的）transport属性，同时置（同样是Protocol的）connected属性为True，正如图9描述的一样：

![pic 9](http://s4.sinaimg.cn/middle/704b6af749e993f5e6453&690)

图9：Protocol遇到其Transport

一旦初始化到这一步后，Protocol开始其真正的工作—将低层的数据流翻译成高层的协议规定格式的消息。处理接收到数据的主要方法是dataReceived，我们的客户端是这样实现的：

    def dataReceived(self, data):
        self.poem += data
        msg = 'Task %d: got %d bytes of poetry from %s'
        print  msg % (self.task_num, len(data), self.transport.getHost())

每次dateReceved被调用就意味着我们得到一个新字符串。由于与异步I/O交互，我们不知道能接收到多少数据，因此将接收到的数据缓存下来直到完成一个完整的协议规定格式的消息。在我们的例子中，诗歌只有在连接关闭时才下载完毕，因此我们只是不断地将接收到的数据添加到我们的.poem属性字段中。

注意我们使用了Transport的getHost方法来取得数据来自的服务器信息。我们这样做只是与前面的客户端保持一致。相反，我们的代码没有必要这样做，因为我们没有向服务器发送任何消息，也就没有必要知道服务器的信息了。

我们来看一下dataReceved运行时的快照。在2.0版本相同的目录下有一个twisted-client-2/get-poetry-stack.py。它与2.0版本的不同之处只在于：

    def dataReceived(self, data):
        traceback.print_stack()
        os._exit(0)

这样一改，我们就能打印出跟踪堆栈的信息，然后离开程序，可以用下面的命令来运行它：

    python twisted-client-2/get-poetry-stack.py 10000

你会得到内容如下的跟踪堆栈：

    File "twisted-client-2/get-poetry-stack.py", line 125, in
    poetry_main() 
      ... # I removed a bunch of lines here 
      File ".../twisted/internet/tcp.py", line 463, in doRead # Note the doRead callback 
    return self.protocol.dataReceived(data) 
    File "twisted-client-2/get-poetry-stack.py", line 58, in dataReceived traceback.print_stack()
    
看见没，有我们在1.0版本客户端的doRead回调函数。我们前面也提到过，Twisted在建立新抽象层进会使用已有的实现而不是另起炉灶。因此必然会有一个IReadDescriptor的实例在辛苦的工作，它是由Twisted代码而非我们自己的代码来实现。如果你表示怀疑，那么就看看twisted.internet.tcp中的实现吧。如果你浏览代码会发现，由同一个类实现了IWriteDescriptor与ITransport。因此 IreadDescriptor实际上就是变相的Transport类。可以用图10来形象地说明dateReceived的回调过程： 

![pic 10](http://s13.sinaimg.cn/middle/704b6af749e9944af0fec&690)

图10：dateReceived回调过程 
 
一旦诗歌下载完成，PoetryProtocol就会通知它的PooetryClientFactory： 

    def connectionLost(self, reason):     
        self.poemReceived(self.poem) 
    def poemReceived(self, poem):    
        self.factory.poem_finished(self.task_num, poem)

当transport的连接关闭时，conncetionLost回调会被激活。reason参数是一个twisted.python.failure.Failure的实例对象，其携带的信息能够说明连接是被安全的关闭还是由于出错被关闭的。我们的客户端因认为总是能完整地下载完诗歌而忽略了这一参数。

工厂会在所有的诗歌都下载完毕后关闭reactor。再次重申：我们代码的工作就是用来下载诗歌-这意味我们的PoetryClientFactory缺少复用性。我们将在下一部分修正这一缺陷。值得注意的是，poem_finish回调函数是如何通过跟踪剩余诗歌数的：

    ...
    self.poetry_count -= 1
    
    if self.poetry_count == 0:
    …

如果我们采用多线程以让每个线程分别下载诗歌，这样我们就必须使用一把锁来管理这段代码以免多个线程在同一时间调用poem_finish。但是在交互式体系下就不必担心了。由于reactor只能一次启用一个回调。

新的客户端实现在处理错误上也比先前的优雅的多，下面是PoetryClientFactory处理错误连接的回调实现代码：

    def clientConnectionFailed(self, connector, reason):
        print 'Failed to connect to:', connector.getDestination()
        self.poem_finished()
    
注意，回调是在工厂内部而不是协议内部实现。由于协议是在连接建立后才创建的，而工厂能够在连接未能成功建立时捕获消息。

**结束语：**

版本2的客户端使用的抽象对于那些Twisted高手应该非常熟悉。如果仅仅是为在命令行上打印出下载的诗歌这个功能，那么我们已经完成了。但如果想使我们的代码能够复用，能够被内嵌在一些包含诗歌下载功能并可以做其它事情的大软件中，我们还有许多工作要做，我们将在第六部分讲解相关内容。

#####第六部分：抽象地利用Twisted

**打造可以复用的诗歌下载客户端**

我们在实现客户端上已经花了大量的工作。最新版本的（2.0）客户端使用了Transports，Protocols和Protocol Factories，即整个Twisted的网络框架。但仍有大的改进空间。2.0版本的客户端只能在命令行里下载诗歌。这是因为PoetryClientFactory不仅要下载诗歌还要负责在下载完毕后关闭程序。但这对于 `PeotryClientFactory` 的确是一项分外的工作，因为它除了做好生成一个PoetryProtocol的实例和收集下载完毕的诗歌的工作外最好什么也别做。

我需要一种方式来将诗歌传给开始时请求它的函数。在同步程序中我们会声明这样的API:

    def get_poetry(host, post):
        """Return a poem from the poetry server at the given host and port."""

当然了，我们不能这样做。诗歌在没有全部下载完前上面的程序是需要被阻塞的，否则的话，就无法按照上面的描述那样去工作。但是这是一个交互式的程序，因此对于阻塞在socket是不会允许的。我们需要一种方式来告诉调用者何时诗歌下载完毕，无需在诗歌传输过程中将其阻塞。这恰好又是Twisted要解决的问题。Twisted需要告诉我们的代码何时socket上可以读写、何时超时等等。我们前面已经看到Twisted使用回调机制来解决问题。因此，我们也可以使用回调：

    def get_poetry(host, port, callback):
        """
        Download a poem from the given host and port and invoke
     
          callback(poem)
     
        when the poem is complete.
        """

现在我们有一个可以与Twisted一起使用的异步API，剩下的工作就是来实现它了。

前面说过，我们有时会采用非Twisted的方式来写我们的程序。这是一次。你会在第七和八部分看到真正的Twisted方式（当然，它使用了抽象）。先简单点讲更晚让大家明白其机制。

**客户端3.0**

可以在twisted-client-3/get-poetry.py看到3.0版本。这个版本实现了get_poetry方法：

    def get_poetry(host, port, callback):
        from twisted.internet import reactor
        factory = PoetryClientFactory(callback)
        reactor.connectTCP(host, port, factory)

这个版本新的变动就是将一个回调函数传递给了PoetryClientFactory。这个Factory用这个回调来将下载完毕的诗歌传回去。

    class PoetryClientFactory(ClientFactory):
        protocol = PoetryProtocol
        
        def __init__(self, callback):
            self.callback = callback
        
        def poem_finished(self, poem):
            self.callback(poem)

值得注意的是，这个版本中的工厂因其不用负责关闭reactor而比2.0版本的简单多了。它也将处理连接失败的工作除去了，后面我们会改正这一点。PoetryProtocol无需进行任何变动，我们就直接复用2.1版本的：

    class PoetryProtocol(Protocol):
        poem = ''
        def dataReceived(self, data):
            self.poem += data
        def connectionLost(self, reason):
            self.poemReceived(self.poem)
        def poemReceived(self, poem):
            self.factory.poem_finished(poem)

通过这一变动，get_poetry,PoetryClientFactory与PoetryProtocol类都完全可以复用了。它们都仅仅与诗歌下载有关。所有启动与关闭reactor的逻辑都在main中实现：

def poetry_main():
    addresses = parse_args()
    from twisted.internet import reactor
    poems = []

    def got_poem(poem):
        poems.append(poem)
        if len(poems) == len(addresses):
            reactor.stop()

    for address in addresses:
        host, port = address
        get_poetry(host, port, got_poem)

    reactor.run()

    for poem in poems:
        print poem

因此，只要我们需要，就可以将这些可复用部分放在任何其它想实现下载诗歌功能的模块中。
顺便说一句，当你测试3.0版本客户端时，可以重配置诗歌下载服务器来使用诗歌下载的快点。现在客户端下载的速度就不会像前面那样让人”应接不暇“了。

讨论
我们可以用图11来形象地展示回调的整个过程：

![pic 10](http://krondo.com/blog/wp-content/uploads/2009/09/reactor-poem-callback.png)

图10 ：回调过程

图10是值得好好思考一下的。到现在为止，我们已经完整描绘了一个一直到向我们的代码发出信号的整个回调链条。但当你用Twisted写程序时，或其它交互式的系统时，这些回调中会包含一些我们的代码来回调其它的代码。换句话说，交互式的编程方式不会在我们的代码处止步（Dave的意思是说，我们的回调函数中可能还会回调其它别人实现的代码，即交互方式不会止步于我们的代码，这个方式会继续深入到框架的代码或其它第三方的代码）。

当你在选择Twisted实现你的工程时，务必记住下面这几条。当你作出决定：

    I'm going to use Twisted!

即代表你已经作出这样的决定：

我将要构造我的程序如由reactorz牵引的一系列的异步回调链

现在也许你还不会像我一样大声地喊出，但它确实是这样的。那就是Twisted的工作方式。

貌似大部分Python程序与Python模块都是同步的。如果我们正在写一个同样需要下载诗歌的同步方式的程序，我可能会通过在我们的代码中添加下面几句来实现我们的同步方式的下载诗歌客户端版本：

    ...
    import poetrylib # I just made this module name up
    poem = poetrylib.get_poetry(host, port)
    ...

然后我们继续。如果我们决定不需要这个这业务那我们可以将这几行代码去掉就OK了。如果我们真的要用Twisted版本的get_poetry来实现同步程序，那么我们需要对异步方式中的回调进行大的改写。这里，我并不想说改写程序不好。而是想说，简单地将同步与异步的程序混合在一直是不行的。

如果你是一个Twisted新手或初次接触异步编程，建议你在试图复用其它异步代码时先写点异步Twisted的程序。这样你不用去处理因需要考虑各个模块交互关系而带来的复杂情况下，感受一下Twisted的运行机制。

如果你的程序原来就是异步方式，那么使用Twisted就再好不过了。Twisted与pyGTK和pyQT这两个基于reactor的GUI工具包实现了很好的可交互性。

**异常问题的处理**

在版本3.0中，我们没有去检测与服务器的连接失败的情况，这比在1.0版本中出现时带来的麻多得多。如果我们让3.0版本的客户端到一个不存在的服务器上下载诗歌，那么不是像1.0版本那样立刻程序崩溃掉而是永远处于等待状态中。clientConncetionFailed回调仍然会被调用，但是因为其在ClientFactory基类中什么也没有实现（若子类没有重写基类函数则使用基类的函数）。因此，got_poem回调将永远不会被激活，这样一来，reactor也不会停止了。我们已经在第2部分也遇到过这样一个不做任何事情的函数了。

因此，我们需要解决这一问题，在哪儿解决呢？连接失败的信息会通过clientConnectionFailed函数传递给工厂对象，因此我们就从这个函数入手。但这个工厂是需要设计成可复用的，因此如何合理处理这个错误是依赖于工厂所使用的场景的。在一些应用中，丢失诗歌是很糟糕的;但另外一些应用场景下，我们只是尽量尝试，不行就从其它地方下载 。换句话说，使用get_poetry的人需要知道会在何时出现这种问题，而不仅仅是什么情况下会正常运行。在一个同步程序中，get_poetry可能会抛出一个异常并调用含有try/excep表达式的代码来处理异常。但在一个异步交互的程序中，错误信息也必须异步的传递出去。总之，在取得get_poetry之前，我们是不会发现连接失败这种错误的。下面是一种可能：

    def get_poetry(host, port, callback):
        """
        Download a poem from the given host and port and invoke
     
          callback(poem)
     
        when the poem is complete. If there is a failure, invoke:
     
          callback(None)
     
        instead.
        """

通过检查回调函数的参数来判断我们是否已经完成诗歌下载。这样可能会避免客户端无休止运行下去的情况发生，但这样做仍会带来一些问题。首先，使用None来表示失败好像有点牵强。一些异步的API可能会将None而不是错误状态字作为默认返回值。其次，None值所携带的信息量太少。它不能告诉我们出的什么错，更不说可以在调试中为我呈现出一个跟踪对象了。好的，也可以尝试这样：

def get_poetry(host, port, callback):
    """
    Download a poem from the given host and port and invoke
 
      callback(poem)
 
    when the poem is complete. If there is a failure, invoke:
 
      callback(err)
 
    instead, where err is an Exception instance.
    """

使用Exception已经比较接近于我们的异步程序了。现在我们可以通过得到Exception来获得相比得到一个None多的多的出错信息了。正常情况下，在Python中遇到一个异常会得到一个跟踪异常栈以让我们来分析，或是为了日后的调试而打印异常信息日志。跟踪栈相当重要的，因此我们不能因为使用异步编程就将其丢弃。
记住，我们并不想在回调激活时打印跟踪栈，那并不是出问题的地方。我们想得到是Exception实例用其被抛出的位置。

Twisted含有一个抽象类称作Failure，如果有异常出现的话，其能捕获Exception与跟踪栈。

Failure的描述文档说明了如何创建它。将一个Failure对象付给回调函数，我们就可以为以后的调试保存跟踪栈的信息了。
在twisted-failure/failure-examples.py中有一些使用Failure对象的示例代码。它演示了Failure是如何从一个抛出的异常中保存跟踪栈信息的，即使在except块外部。我不用在创建一个Failure上花太多功夫。在第七部分中，我们将看到Twisted如何为我们完成这些工作。好了，看看下面这个尝试：

    def get_poetry(host, port, callback):
        """
        Download a poem from the given host and port and invoke
          callback(poem)
        when the poem is complete. If there is a failure, invoke:
          callback(err)
         instead, where err is a twisted.python.failure.Failure instance.
        """

在这个版本中，我们得到了Exception和出现问题时的跟踪栈。这已经很不错了！
大多数情况下，到这个就OK了，但我们曾经遇到过另外一个问题。使用相同的回调来处理正常的与不正常的结果是一件莫名奇妙的事。通常情况下，我们在处理失败信息进，相比成功信息要进行不同的操作。在同步Python编程中，我们经常在处理失败与成功两种信息上采用不同的处理路径，即try/except处理方式：

    try:
        attempt_to_do_something_with_poetry()
    except RhymeSchemeViolation:
        # the code path when things go wrong
    else:
        # the code path when things go so, so right baby

如果我们想保留这种错误处理方式，那么我们需要独立的代码来处理错误信息。那么在异步方式中，这就意味着一个独立的回调：

    def get_poetry(host, port, callback, errback):
        """
        Download a poem from the given host and port and invoke
          callback(poem)
        when the poem is complete. If there is a failure, invoke:
          errback(err)
        instead, where err is a twisted.python.failure.Failure instance.
        """

**版本3.1**

版本3.1实现位于twisted-client-3/get-poetry-1.py。改变是很直观的。PoetryClientFactory，获得了callback和errback两个回调，并且其中我们实现了clientConnectFailed：

    class PoetryClientFactory(ClientFactory):
        protocol = PoetryProtocol
        def __init__(self, callback, errback):
            self.callback = callback
            self.errback = errback 
        def poem_finished(self, poem):
            self.callback(poem)
        def clientConnectionFailed(self, connector, reason):
            self.errback(reason)

由于clientConncetFailed已经收到一个Failure对象（其作为reason参数）来解释为什么会发生连接失败，我们直接将其交给了errback回调函数。直接运行3.1版本（无需开启诗歌下载服务）的代码你会得到如下输出：

    Poem failed: [Failure instance: Traceback (failure with no frames): : Connection was refused by other side: 111: Connection refused. ]

这是由poem_failed回调中的print函数打印出来的。在这个例子中，Twisted只是简单将一个Exception传递给了我们而没有抛出它，因此这里我们并没有看到跟踪栈。因为这并不一个Bug，所以跟踪栈也不需要，Twisted只是想通知我们连接出错。

**总结：**

我们在第六部分学到：

我们为Twisted程序写的API必须是异步的  
不能将同步与异步代码混合起来使用  
我们可以在自己的代码中写回调函数，正如Twisted做的那样  
并且，我们需要写处理错误信息的回调函数  

使用Twisted时，难道在写我们自己的API时都要额外的加上两个参数：正常的回调与出现错误时的回调。幸运的是，Twisted使用了一种机制来解决了这一问题，我们将在第七部分学习这部分内容。

#####第七部分：小插曲，Deferred

**回调函数的后序发展**

在第六部分我们认识这样一个情况：回调是Twisted异步编程中的基础。除了与reactor交互外，回调可以安插在任何我们写的Twisted结构内。因此在使用Twisted或其它基于reactor的异步编程体系时，都意味需要将我们的代码组织成一系列由reactor循环可以激活的回调函数链。

即使一个简单的get_poetry函数都需要回调，两个回调函数中一个用于处理正常结果而另一个用于处理错误。作为一个Twisted程序员，我们必须充分利用这一点。应该花点时间思考一下如何更好地使用回调及使用过程中会遇到什么困难。

分析下3.1版本中的get_poetry函数：

    ...
    def got_poem(poem):
        print poem
        reactor.stop()
    def poem_failed(err):
        print >>sys.stderr, 'poem download failed'
        print >>sys.stderr, 'I am terribly sorry'
        print >>sys.stderr, 'try again later?'
        reactor.stop()

    get_poetry(host, port, got_poem, poem_failed)
 
    reactor.run()
    
我们想法很简单：

1. 如果完成诗歌下载，那么就打印它
2. 如果没有下载到诗歌，那就打印出错误信息
3. 上面任何一种情况出现，都要停止程序继续运行

同步程序中处理上面的情况会采用如下方式：

    ...
    try:
        poem = get_poetry(host, port) # the synchronous version of get_poetry
    except Exception, err:
        print >>sys.stderr, 'poem download failed'
        print >>sys.stderr, 'I am terribly sorry'
        print >>sys.stderr, 'try again later?'
        sys.exit()
    else:
        print poem
        sys.exit()

即callback类似else处理路径，而errback类似except处理路径。这意味着激活errback回调函数类似于同步程序中抛出一个异常，而激活一个callback意味着同步程序中的正常执行路径。

两个版本有什么不同之外吗？可以明确的是，在同步版本中，Python解释器可以确保只要get_poetry抛出何种类型的异步都会执行except块。即只要我们相信Python解释器能够正确的解释执行Python程序，那么就可以相信异常处理块会在恰当的时间点被执行。

不异步版本相反的是：poem_failed错误回调是由我们自己的代码激活并调用的，即PeotryClientFactory的clientConnectFailed函数。是我们自己而不是Python来确保当出错时错误处理代码能够执行。因此我们必须保证通过调用携带Failure对象的errback来处理任何可能的错误。

否则，我们的程序就会因为等待一个永远不会出现的回调而止步不前。

这里显示出了同步与异步版本的又一个不同之处。如果我们在同步版本中没有使用try/except捕获异步，那么Python解释器会为我们捕获然后关掉我们的程序并打印出错误信息。但是如果我们忘记抛出我们的异步异常（在本程序中是在PoetryClientFactory调用errback），我们的程序会一直运行下去，还开心地以为什么事都没有呢。
显而易见，在异步程序中处理错误是相当重要的，甚至有些严峻。也可以说在异步程序中处理错误信息比处理正常的信息要重要的多，这是因为错误会以多种方式出现，而正确的结果出现的方式是唯一的。当使用Twisted编程时忘记处理异常是一个常犯的错误。

关于上面同步程序代码的另一个默认实事是：else与except块两者只能是运行其中一个（假设我们的get_poetry没有在一个无限循环中运行）。Python解释器不会突然决定两者都运行或突发奇想来运行else块27次。对于通过Python来实现那样的动作是不可能的。

但在异步程序中，我们要负责callback和errback的运行。因此，我们可能就会犯这样的错误：同时调用了callback与errback或激活callback27次。这对于使用get_poetry的用户来说是不幸的。虽然在描述文档中没有明确地说明，像try/except块中的else与except一样，对于每次调用get_poetry时callback与errback只能运行其中一个，不管是我们是否成功得下载完诗歌。

设想一下，我们在调试某个程序时，我们提出了三次诗歌下载请求，但是得到有7次callback被激活和2次errback被激活。可能这时，你会下来检查一下，什么时候get_p
oetry激活了两次callback并且还抛出一个错误出来。
从另一个视角来看，两个版本都有代码重复。异步的版本中含有两次reactor.stop，同步版本中含有两次sys.exit调用。我们可以重构同步版本如下：

    ...
    try:
        poem = get_poetry(host, port) # the synchronous version of get_poetry
    except Exception, err:
        print >>sys.stderr, 'poem download failed'
        print >>sys.stderr, 'I am terribly sorry'
        print >>sys.stderr, 'try again later?'
    else:
        print poem
    
    sys.exit()
    
我们可以以同样的方式来重构异步版本吗？说实话，确实不太可能，因为callback与errback是两个不同的函数。难道要我们回到使用单一回调来实现重构吗？
好下面是我们在讨论使用回调编程时的一些观点：

1. 激活errback是非常重要的。由于errback的功能与except块相同，因此用户需要确保它们的存在。他们并不可选项，而是必选项。
2. 不在错误的时间点激活回调与在正确的时间点激活回调同等重要。典型的用法是，callback与errback是互斥的即只能运行其中一个。
3. 使用回调函数的代码重构起来有些困难。

来下面的部分，我们还会讨论回调，但是已经可以明白为什么Twisted引入了deferred抽象机制来管理回调了。

**Deferred**

由于架设在异步程序中大量被使用，并且我们也看到了，正确的使用这一机制需要一些技巧。因此，Twisted开发者设计了一种抽象机制-Deferred-以让程序员在使用回调时更简便。

一个Deferred有一对回调链，一个是为针对正确结果，另一个针对错误结果。新创建的Deferred的这两条链是空的。我们可以向两条链里分别添加callback与errback。其后，就可以用正确的结果或异常来激活Deferred。激活Deferred意味着以我们添加的顺序激活callback或errback。图12展示了一个拥有callback/errback链的Deferred对象：

![pic 12](http://s4.sinaimg.cn/middle/704b6af749ed2370a6a33&690)

图12： Deferred

由于defered中不使用reactor，所以我们可以不用在事件循环中使用它。也许你在Deferred中发现一个seTimeout的函数中使用了reactor。放心，它在将来的版本中删掉。

下面是我们第一次使用deferred的例子twisted-deferred/defer-1.py:

    from twisted.internet.defer import Deferred
     
    def got_poem(res):
        print 'Your poem is served:'
        print res
     
    def poem_failed(err):
        print 'No poetry for you.'
     
    d = Deferred()
     
    # add a callback/errback pair to the chain
    d.addCallbacks(got_poem, poem_failed)
     
    # fire the chain with a normal result
    d.callback('This poem is short.')
     
    print "Finished"

代码开始创建了一个新deferred，然后使用addCallbacks添加了callback/errback对，然后使用callback函数激活了其正常结果处理回调链。当然了，由于只含有一个回调函数还算不上链，但不要紧，运行它：

    Your poem is served:
    This poem is short.
    Finished

有几个问题需要注意：

1. 正如3.1版本中我们使用的callback/errback对，添加到deferred中的回调函数只携带一个参数，正确的结果或出错信息。其实，deferred支持回调函数可以有多个参数，但至少得有一个参数并且第一个只能是正确的结果或错误信息。
2. 我们向deferred添加的是回调函数对
3. callbac函数携带仅有的一个参数即正确的结果来激活deferred
4. 从打印结果顺序可以看出，激活的deferred立即调用了回调。没有任何异步的痕迹。这是因为没有reactor参与导致的。

好了，让我们来试试另外一种情况，twisted-deferred/defer-2.py激活了错误处理回调：

    from twisted.internet.defer import Deferred
    from twisted.python.failure import Failure
     
    def got_poem(res):
        print 'Your poem is served:'
        print res
     
    def poem_failed(err):
        print 'No poetry for you.'
     
    d = Deferred()
     
    # add a callback/errback pair to the chain
    d.addCallbacks(got_poem, poem_failed)
     
    # fire the chain with an error result
    d.errback(Failure(Exception('I have failed.')))
     
    print "Finished"

运行它打印出的结果为：

    No poetry for you.
    Finished
    
激活errback链就调用errback函数而不是callback，并且传进的参数也是错误信息。正如上面那样，errback在deferred激活就被调用。
在前面的例子中，我们将一个Failure对象传给了errback。deferred会将一个Exception对象转换成Failure，因此我们可以这样写：

    from twisted.internet.defer import Deferred
     
    def got_poem(res):
        print 'Your poem is served:'
        print res
     
    def poem_failed(err):
        print err.__class__
        print err
        print 'No poetry for you.'
     
    d = Deferred()
     
    # add a callback/errback pair to the chain
    d.addCallbacks(got_poem, poem_failed)
     
    # fire the chain with an error result
    d.errback(Exception('I have failed.'))

运行结果如下：

    twisted.python.failure.Failure [Failure instance: Traceback (failure with no frames): : I have failed. ]
    No poetry for you.

这意味着在使用deferred时，我们可以正常地使用Exception。其中deferred会为我们完成向Failure的转换。
下面我们来运行下面的代码看看会出现什么结果：

    from twisted.internet.defer import Deferred
    def out(s): print s
    d = Deferred()
    d.addCallbacks(out, out)
    d.callback('First result')
    d.callback('Second result')
    print 'Finished'

输出结果：
    
    First result Traceback (most recent call last): ... twisted.internet.defer.AlreadyCalledError
    
很意外吧，也就是说deferred不允许别人激活它两次。这也就解决了上面出现的那个问题：一个激活会导致多个回调同时出现。而deferred设计机制控制住了这种可能，如果你非要在一个deferred上要激活多个回调，那么正如上面那样，会报异常错。

那deferred能帮助我们重构异步代码吗？考虑下面这个例子：

    import sys
     
    from twisted.internet.defer import Deferred
     
    def got_poem(poem):
        print poem
        from twisted.internet import reactor
        reactor.stop()
     
    def poem_failed(err):
        print >>sys.stderr, 'poem download failed'
        print >>sys.stderr, 'I am terribly sorry'
        print >>sys.stderr, 'try again later?'
        from twisted.internet import reactor
        reactor.stop()
     
    d = Deferred()
     
    d.addCallbacks(got_poem, poem_failed)
     
    from twisted.internet import reactor
     
    reactor.callWhenRunning(d.callback, 'Another short poem.')
     
    reactor.run()

这基本上与我们上面的代码相同，唯一不同的是加进了reactor。我们在启动reactor后调用了callWhenRunning函数来激活deferred。我们利用了callWhenRunning函数可以接收一个额外的参数给回调函数。多数Twisted的API都以这样的方式注册回调函数，包括向deferred添加callback的API。下面我们给deferred回调链添加第二个回调：
    
    import sys
     
    from twisted.internet.defer import Deferred
     
    def got_poem(poem):
        print poem
     
    def poem_failed(err):
        print >>sys.stderr, 'poem download failed'
        print >>sys.stderr, 'I am terribly sorry'
        print >>sys.stderr, 'try again later?'
     
    def poem_done(_):
        from twisted.internet import reactor
        reactor.stop()
     
    d = Deferred()
     
    d.addCallbacks(got_poem, poem_failed)
    d.addBoth(poem_done)
     
    from twisted.internet import reactor
     
    reactor.callWhenRunning(d.callback, 'Another short poem.')
     
    reactor.run()
    
addBoth函数向callback与errback链中添加了相同的回调函数。在这种方式下，deferred有可能也会执行errback链中的回调。这将在下面的部分讨论，只要记住后面我们还会深入讨论deferred。

总结：

在这部分我们分析了回调编程与其中潜藏的问题。我们也认识到了deferred是如何帮我们解决这些问题的：

1. 我们不能忽视errback，在任何异步编程的API中都需要它。Deferred支持errbacks。
2. 激活回调多次可能会导致很严重的问题。Deferred只能被激活一次，这就类似于同步编程中的try/except的处理方法。
3. 含有回调的程序在重构时相当困难。有了deferred，我们就通过修改回调链来重构程序。

关于deferred的故事还没有结束，后面还有大量的细节来讲。但对于使用它来重构我们的客户端已经够用的了，在第八部分将讲述这部分内容。

#####第八部分：使用Deferred的诗歌下载客户端


**客户端4.0**

我们已经对deferreds有些理解了，现在我们可以使用它重写我们的客户端。你可以在twisted-client-4/get-poetry.py中看到它的实现。

这里的get_poetry已经再也不需要callback与errback参数了。相反，返回了一个用户可能根据需要添加callbacks和errbacks的新deferred。

    def get_poetry(host, port):
        """
        Download a poem from the given host and port. This function
        returns a Deferred which will be fired with the complete text of
        the poem or a Failure if the poem could not be downloaded.
        """
        d = defer.Deferred()
        from twisted.internet import reactor
        factory = PoetryClientFactory(d)
        reactor.connectTCP(host, port, factory)
        return d

这里的工厂使用一个deferred而不callback/errback对来初始化。一旦我们获取到poem后或者没有连接到服务器上，deferred就会以返回一首诗歌或一个failure的被激活。
 
    class PoetryClientFactory(ClientFactory): 
        protocol = PoetryProtocol 
        def __init__(self, deferred):
            self.deferred = deferred
        def poem_finished(self, poem):
            if self.deferred is not None：
                d, self.deferred = self.deferred, None 
                d.callback(poem)
        def clientConnectionFailed(self, connector, reason)：
            if self.deferred is not None：
                d, self.deferred = self.deferred, None
                d.errback(reason)
         
注意我们在deferred被激活后是如何销毁其引用的。这种方式普便存在于Twisted的源代码中，这样做可以保证我们不会激活一个deferred两次。这也为Python的垃圾回收带来的方便。

这里仍然不用去改变poetryProtocol。我们只需要更新poetry_main函数即可：

    def poetry_main(): 
        addresses = parse_args()
        from twisted.internet import reactor
        poems = [] 
        errors = [] 

        def got_poem(poem):
            poems.append(poem) 
            
        def poem_failed(err):
            print >>sys.stderr, 'Poem failed:', err
            errors.append(err) 
            
        def poem_done(_):
            if len(poems) + len(errors) == len(addresses):
                reactor.stop()
                
        for address in addresses:
            host, port = address 
            d = get_poetry(host, port) 
            d.addCallbacks(got_poem, poem_failed)
            d.addBoth(poem_done) 
            reactor.run() 
    
        for poem in poems: 
            print poem

注意我们是如何利用deferred的回调链在不考虑两个主要的callback与errback回调外，重构poem_done调用的。
由于deferred在Twisted大量被使用，使用小写字母d来表示当前正在工作中的deferred已经成为惯例。

**讨论**

新版本的客户端与我们前面的同步版本的客户端一样，get_poetry得到的参数都是诗歌下载服务器的地址。同步版本返回的是诗歌内容，而异步版本返回的却是一个deferred。返回一个deferred是Twisted的APIs或用Twisted写的程序常见的，这样一来我们可以这样来理解deferred：

一个Deferred代表了一个“异步的结果”或者“结果还没有到来”

在图13中可以更加清晰地表达出两者之间的不同：

![pic 13](http://krondo.com/blog/wp-content/uploads/2009/10/sync-async.png)

图13：同步 VS 异步

异步函数返回一个deferred，对用户意味着：

我是一个异步函数。不管你想要什么，可能现在马上都得不到。但当结果来到时，我会激活这个deferred的callback链并返回结果。或者当出错时，相应地激活errback链并返回出错信息。

当然，这个函数是不能随意激活这个deferred的，因为它已经返回了。但这个函数已经启动了一系列事件，这些事件最终将会激活这个deferred。

因此，deferred是为适应异步模式的一种延迟函数返回的方式。函数返回一个deferred意味着其是异步的，代表着将来的结果，也是对将来能够返回结果的一种承诺。

同步函数也能返回一个deferred，因此严格点说，返回deferred只说可能是异步的。我们会在将来的例子中看到同步函数返回deferred。

由于deferred的行为已经很好的定义与理解，因此在实现自己的API时返回一个deferred更容易让其它的Twisted程序理解你的代码。如果没有deferred，可能每个人写的模块都使用不同的方式来处理回调。这要一来就增加了相互理解的工作量。

当你使用Deferred时，你仍然在使用回调，它们仍然由reactor来调用。

当首次学习Twisted时，经常犯的一个错误就是：会给deferred增加一些它本身不能实现的功能。尤其是：经常假设在deferred上添加一个函数就可以使其变成异步函数。这可能会让你产生这样的想法：在Twisted 中可以通过将os.system的函数添加到deferred的回调链中。

我认为，这可能是没有弄清楚异步编程的原因才产生这样的想法。由于Twisted代码使用了大量的deferred但却很少会涉及到reactor，可能会认为deferred做了大部分工作。如果你是从开始阅读这个系列的，你就会知道事情远不是这样。虽然Twisted是由众多部分组合在一起来工作的，但实现异步的主要工作都是由reactor来完成的。Deferred是一个很好的抽象概念，但前面几个例子中的客户端我们却没有使用它，而reactor却都用到了。

来看看我们第一个回调激活时的跟踪栈信息。运行twisted-client-4/get-poetry-stack.py让其连接你打开的服务器：

    File "twisted-client-4/get-poetry-stack.py", line 129, in
    poetry_main()
    File "twisted-client-4/get-poetry-stack.py", line 122, in poetry_main
    reactor.run() ... # some more Twisted function calls
    protocol.connectionLost(reason) 
    File "twisted-client-4/get-poetry-stack.py", line 59, in connectionLost 
    self.poemReceived(self.poem) 
    File "twisted-client-4/get-poetry-stack.py", line 62, in poemReceived
    self.factory.poem_finished(poem)
    File "twisted-client-4/get-poetry-stack.py", line 75, in poem_finished
    d.callback(poem) # here's where we fire the deferred 
    ... # some more methods on Deferreds
    File "twisted-client-4/get-poetry-stack.py", line 105, in got_poem
    traceback.print_stack()

这很像版本2.0的跟踪栈，图14可以很好地说明具体的调用关系：

![pic 14](http://krondo.com/blog/wp-content/uploads/2009/10/reactor-deferred-callback.png)

图14 deferred的回调

这很类似于我们前面的Twisted客户端，虽然这张图的调用关系并不清晰而会你摸不着头脑。但我们先不深入分析这张图。有一个细节并没有在这张图上反映出来：callback链直到第二个回调poem_done激活前才将控制权还给reactor。

通过使用deferred，我们在由Twisted中的reactor启动的回调中加入了一些自己的东西，但我们并没有改变异步程序的基础架构。回忆下回调编程的特点：

1. 在一个时刻，只会有一个回调在运行
2. 当reactor运行时，那我们自己的代码则得不到运行
3, 反之则反之
4. 如果我们的回调函数发生阻塞，那么整个程序就跟着阻塞掉了

在一个 deferred上追加一个回调并不会改变上面这些实事。尤其是，第4 条。因此当一个deferred激活时被阻塞，那么整个Twisted就会陷入阻塞中。因此我们会得到如下结论：

Deferred只是解决回调函数管理问题的一种解决方案。它并不一种替代回调方式也不能将阻塞式的回调变成非阻塞式回调的。

我通过构建一个添加阻塞式回调的deferred来验证最后一点。验证代码文件为twisted-deferred/defer-block.py。第二个callback通过使用time.sleep来达到阻塞的效果。如果你运行该代码来观察打印信息顺序时，你会发现deferred中阻塞回调仍然会阻塞掉。

**总结**

函数通过返回一个Deferred，向使用者暗示“我是采用异步方式的”并且当结果到来时会使用一种特殊的机制（在此处添加你的callback与errback）来获得返回结果。Defered被广泛地运用在Twisted的每个角落，当你浏览Twisted源码时你就会不停地遇到它。

4.0版本客户端是第一个使用Deferred的Twisted版的客户端，其使用方法为在其异步函数中返回一个deferred来。可以使用一些Twisted的APIs来使客户端的实现更加清晰些，但我觉得它能够很好地体现出一个简单的Twisted程序是怎么写的了，至少对于客户端可以如此肯定。事实上，后面我们会重构我们的服务器端。

但我们对Deferred的讲解还没有结束。使用如此少量的代码，Deferred就能提供如此之多的功能。我们将在第9部分探讨其更多的功能和功能背后的动机。

#####第九部分：第二个小插曲，Deferred


**更多关于回调的知识**

稍微停下来再思考一下回调的机制。尽管对于以Twisted方式使用Deferred写一个简单的异步程序已经非常了解了，但Deferred提供更多的是只有在比较复杂环境下才会用到的功能。因此，下面我们自己想出一些复杂的环境，以此来观察当使用回调编程时会遇到哪些问题。然后，再来看看deferred是如何解决这些问题的。
因此，我们为诗歌下载客户端添加了一个假想的功能。设想一些计算机科学家发明了一种新诗歌关联算法，
Byronification引擎。这个漂亮的算法根据一首诗歌生成一首使用Lord Byron式的同样的诗歌。另外，专家们提供了其Python的接口，即：

    class IByronificationEngine(Interface): 
        def byronificate(poem):
            """
            Return a new poem like the original, but in the style of Lord Byron.
            Raises GibberishError if the input is not a genuine poem.
            """

像大多数高尖端的软件一样，其实现都存在着许多bugs。这意外着除了已知的异常外，这个byronificate 方法可能会抛出一些专家当时没有预料到的异常出来。

我们还可以假设这个引擎能够非常快的动作以至于我们可以在主线程中调用到而无需考虑使用reactor。下面是我们想让程序实现的效果：

1. 尝试下载诗歌
2. 如果下载失败，告诉用户没有得到诗歌
3. 如果下载到诗歌，则转交给Byronificate处理引擎一份
4. 如果引擎抛出GibberishError，告诉用户没有得到诗歌
5. 如果引擎抛出其它异常，则将原始式样的诗歌立给用户
6. 如果我们得到这首诗歌，则打印它
7. 结束程序

这里设计是当遇到GibberishError异常则表示没有得到诗歌，因此我们直接告诉用户下载失败即可。这也许对调试没什么用处，但我们的用户关心的只是我们下载到诗歌没有。另一方面，如果引擎因为一些其它的原因而出现处理失败，那么我们将原始诗歌交给用户。毕竟，有诗歌呈现总比没有好，虽然不是用户想要的Byron样式。

下面是同步模式的代码：

    try:
        poem = get_poetry(host, port) # synchronous get_poetry
    except:
        print >>sys.stderr, 'The poem download failed.'
    else:
        try:
            poem = engine.byronificate(poem)
        except GibberishError:
            print >>sys.stderr, 'The poem download failed.'
        except:
            print poem # handle other exceptions by using the original poem
        else:
            print poem
    sys.exit()

这段代码可能经过一些重构会更加简单，但已经足以说明上面的逻辑流程。我们想升级那些最近使用deferred的客户端来使用这个功能。但这部分内容我准备把它放在第十部分。现在，我们来考虑一下，用版本3.1来实现这个功能，最后一个没有使用deferred的客户端。假设我们无需考虑处理异常，那么只是改变一下got_poem回调即可：

    def got_poem(poem):
        poems.append(byron_engine.byronificate(poem))
        poem_done()
    
    
那么如果byronificate抛出GibberishError异常或其它异常会发生什么呢？看看第六部分的图11,我们可以得到：

1. 这个异常会传播到工厂中的poem_finished回调，即激活got_poem的方法
2. 由于poem_finished并没有捕获这个异常，因此其会传递到protocol中的poemReceive函数
3. 然后来到connectionLost函数，仍然在protocol中
4. 然后就来到Twisted的核心区，最后止步于reactor。

前面已经了解到，reactor会捕获异常并记录它而不是“崩溃”掉。但它却不会告诉用户我们的诗歌下载失败的消息。reactor并不知道任何诗歌或GibberishErrors的信息，它只是一段被设计成适应所有网络类型的通用代码，即便与诗歌无关的网络服务。（Dave这里想强调的是reactor只是做一些具有普遍意义的事情，不会单独去处理特定的问题，例如这里原GibberishErrors异常）

注意异常是如何顺着调用链传递到具有通用性代码区域。并且看到，在got_poem后面任何一步都没有可望以我们客户端的具体要求来处理异常的。这与同步代码中的方式恰恰相反。

图15揭示了一个同步客户端的调用栈：


图15：同步调用栈

main函数是最高层，意味着它可以触及整个程序，它为什么要存在，并且它是如何在整体上表现的。典型的，main函数可以触及到用户在命令行输入想让程序做什么的参数。并且它还有一个特殊的目的：为一个命令行式的客户端打印结果。

socket的connet函数，恰恰相反，其为最低层。它所知道的就是提供到指定地址的连接。它并不知道另一端是什么及我们为什么要进行连接。但connect有通用性，不管你因为何种服务要进行网络连接都可以使用它。

get_poetry在中间，它知道要取一些诗歌，但并不知道如果得不到诗歌会发生什么。因此，从connect抛出的异常会向上传递，从低层的具有通用性的代码区到高层的具有针对性的代码区，直到其传递到知道如何处理这个异常的代码区。

现在，我们再回来看看对3.1版的假想功能的实现。我们在图16里对调用栈进行了分析，当然只是说明了其中关键的函数：



图16 异步调用栈
现在问题非常清晰了：在回调中，低层的代码（reactor）调用高层的代码，其甚至还会调用更高层的代码。因此一旦出现了异常，它并不会立即被其附件（在调用栈中可触及）的代码捕获，当然附近的代码也不可能处理它。由于异常每向上传递一次，就越靠近低层那些更加不知如何处理该异常的代码。

一旦异常来到Twisted的核心代码区，游戏也就结束了。异常并不会被处理，只是被记录下来。因此我们在以最原始的回调方式使用回调时（不使用deferred），必须在其进入Twisted之间很好地处理各种异常，至少是我们知道的那些在我们自己设定的规则下会产生的异常。当然其也应该包括那些由我们自己的BUG产生的异常。

由于bug可能存在于我们代码中的每个角落，因此我们必须将每个回调都放入try/except中，这样一来所有的异常都才有可能被捕获。这对于我们的errback同样适用，因为errback中也可能含有bugs。

Deferred的优秀架构

最终还得由Deferred来帮我们解决这类问题。当一个deferred激活了一个callback或errback时，它就会捕获各种由回调抛出的异常。换句话说，deferred扮演了try/except模块，这样一来，只要我们使用deferred就无需自己来实现这一层了。那deferred是如何解决这个问题的？很简单，它传递异常给在其链上的下一个errback。
我们添加到deferred中的第一个errback回调来处理任何出错信息，信息是在deferred的errback函数调用时发出的。但第二个errback会处理任何由第一个errback或第一个callback抛出的异常，并一直按这种规则传递下去。

回忆下图12.我们假设第一对callback/errback是stage0,下面则是stage1，stage2。。。依次类推。

对于stage N来说，如果其callback或errback出错，那么stage N+1的errback就会被调用并收到一个Failure对象作为参数，同时stage N+1的callback就不会被调用了。

通过将回调函数产生的异常向在链中传递，deferred将异常抛向了高层代码。这也意味着调用deferred的callback与errback永远不会在调用都本身处引发异常（只要你仅激活deferred一次），因此，底层的代码可以放心的激活deferred而无需担心会引发异常。相反，高层代码通过向deferred中添加errback（使用addErrback）来捕获异常。
在同步代码中，异常会在其被捕获而停止传递，那么一个errback如何发出其捕获了异常这一信号呢？同样很简单：不再引发异常。这样一来，执行权就转移到了callback中来。因此对于stage N来说，不管是callback还是errback成功执行而没有抛出异常，那么stage N+1的callback就会被调用，同样，stage N+1的errback就不会被调用了。

我们来总结一下吧：

1. 一个deferred有一个callback/errback对链，它们以添加到deferred中的顺序依次排列
2. stage 0，即第一对errback/callbac，会在deferred激活时调用，具体调用那个看激活deferred的方式，若是通过.errback激活，则调用errback；同样若是通过.callback激活则调用callback。（这里的errback/callback实际是指通过addBoth添加的函数）
3. 如果stage N执行出现异常，则stage N+1的errback被调用，并且其参数即为stage N出现的异常
4. 同样，如果stage N成功，即没有抛出异常，则N+1的callback被调用，其第一个参数为stage N的返回值。

图17更加直观的描述上述操作：


图17：deferred中的控制流程

绿色的线表示callback和errback成功执行没抛出异常，而红线表示出现了异常。这些线不仅说明了控制流程还说明了异常与返回值在链中流动的情况。图17显示了所有deferred能出现的可能路径，但实际只有一条路径会存在。图18显示了一条可能的路径：



图18：可能的deferred激活路线

图18中，deferred的.callback函数被调用了，因此激活了stage 0的callback。这个callback成功的执行而没有抛出异常，因此控制权传给了stage 1的callback。但这个callbac执行失败而抛出异常，因此控制权传给了stage 2的errback。errback成功的处理了异常，而没有再抛出异常，因此控制权传给了stage 3的callback，并且将errback的返回值作为第一个参数传了进来（即stage 3的callback中）。
图18中，可以看出，最后一个stage上的所有的回调出现异常时，都由下一层的errback来捕获并处理，但如果最后一个stage的callback或errback执行失败而抛出异常，怎么办呢？那么这个异常就会成为unhandled（未处理）。
在同步代码中，未处理的异常会导致解释器崩溃，在原始方式使用回调的代码中未处理异常会由reactor捕获并记录下来。那么未处理异常出现在deferred中会怎样呢？让我们来做个试验。运行twisted-deferred/defer-unhandled.py试试。下面是输出：
Finished
Unhandled error in Deferred: Traceback (most recent call last):
...
--- <exception caught here> ---
...
exceptions.Exception: oops

如下几点需要引起我们的注意：
1.最后一个print函数成功执行，意味着程序并没有因为出现未处理异常而崩溃。
2.其只是将跟踪栈打印出来，而没有宕掉解释器
3.跟踪栈的内容告诉我们deferred在何处捕获了异常
4.“’Unhandle”的字符在“Finished”之后出现。
之所以出现第4条是因为，这个消息只有在deferred被垃圾回收时才会打印出来。我们将在下面的部分看到其中的原因。
在同步代码中，我们可以使用raise来重新抛出一个异常而无需其它参数。同样，我们也可以在errback中这样做。deferred通过以下两点来判断callback/errback是否执行成功：
1.callback/errback “raise”一个异常，或
2.callbakc/errback返回一个Failure对象
因为errback的第一个参数就是一个Failure，因此一个errback可以在进行完其处理后可以再次抛出这个Failure。

Callbacks与Errbacks，成对出现
上面讨论内容中的一个问题必须要清楚：你添加callback与errback到一个defered的顺序会决定这个deferred的的整体运行情况。另一个必须搞清楚的是：在一个deferred中callback与errback往往是成对出现。有四个方法可以向一个deferred的回调链中添加callback/errback对：

addCallbacks
addCallback
addErrback
addBoth

很明显的是，第一个与第四个是向链中添加函数对。当然中间两个也向链中添加函数对。AddCallback向链中添加一个显式的callback函数与一个隐式的”pass-through“函数（实在想不出一个对应的词）。一个pass-through函数只是虚设的函数，只将其第一个参数返回。由于errback回调函数的第一个参数是Failure，因此一个“path-through”的errback总是执行“失败”，即将异常传给下个errback回调。

deferred模拟器

这部分内容，没有译。其主要是帮助理解deferred，但你会发现，读其中的代码，根本更好的理解deferred。主要是我还没有理解，嘿嘿。所以就不知为不知吧。







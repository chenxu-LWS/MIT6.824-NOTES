# Lec4 Fault-Tolerant Virtual Machines-PART 2⃣️

## VMware FT综述

论文实现了虚拟机的备份，该系统实现了一台服务器上的**备份虚拟机（Backup）复制另一套服务器上的主虚拟机（Primary）**。通过**复制状态机**的方式实现，即对主虚拟机（Primary）的执行指令进行复制。

## VMware FT Replication

论文介绍了两种复制方式：State Transfer（状态转移），Replicated State Machine（复制状态机）

### 状态转移

如果服务器的两个副本始终保持同步，一旦Primary出现故障，由于Backup有所有的信息，则可以直接接管。这就是状态转移。即**Backup是Primary的完全拷贝**。当Primary故障时，Backup就可以从其保存的最新状态运行，所谓的状态转移就是将Primary的全部状态复制转移到Backup。**每过一段时间，Primary都会对自身内存做一份大的拷贝，通过网络同步到Backup。**

### 复制状态机（论文选择的实现方式✅）

复制状态机不会在不同副本间发送状态，它只是**从Primary将这些客户端的操作指令/操作结果发送给Backup**。人们倾向于这种复制手段，因为并不需要做大副本的全部状态的同步，每次操作相对较小。但缺点是相对复杂。

## VMware FT Theory

VMware虚拟机，实际上是在实际的硬件上实现了一个虚拟机监控器VMM（Virtual machine Monitor），Monitor在同一硬件上模拟出多个虚拟的计算机。在VMM上可以运行一系列不同的OS，每一个都有自己的OS内核和应用程序。

Primary虚拟机和Backup虚拟机分属于两台物理机，这样才能抵御物理机的fail-stop故障。VMware FT的目的就是，确保Primary和Backup两台虚拟机的状态保持一致，并确保在Primary改变时，Backup也同步改变。

假设有一个局域网（LAN  Local Area Network），在局域网内除了Primary和Backup所在的机器，还有一些与Primary和Backup交互的客户端机器。

### Request

当客户端向Primary发送一个网络数据包，VMM会做两个事情：

* 在虚拟机的OS中模拟这个数据包到达的中断，将相应的数据送到Primary虚拟机中的应用程序
* 除此之外，VMM会拷贝一份相同的数据包发送给Backup虚拟机所在的VMM，Backup中的VMM也知道这是用于同步的数据包，因此也会在Backup虚拟机中模拟这个数据包中断。

### Response

收到请求后，Primary虚拟机中的服务会生成响应数据包，通过VMM在虚拟机中的虚拟网卡发出来。之后VMM可以看到该报文，随后将这个**报文发送回客户端**；

同时Backup虚拟机也会生成响应数据包，但Backup所在物理机的VMM知道这是一份Backup，因此会**丢弃这个响应**。

因此最终只有Primary的响应会返回给客户端。

### Log Channel

论文中将Primary到Backup之间同步的数据流通道称为Log Channel。从Primary发到Backup的事件称为Log Event或Log Entry。

### Fault Tolerance

> 如果Primary发生故障

在正常情况下，Backup会每秒钟收到近100次的来自Primary的定时器中断消息；而如果发生了故障Backup将不再收到Log Events。即Backup可以感知到Primary没有进行工作。此时Backup虚拟机就会上线，不再等待来自Primary的Log Events。

同时Backup会在VMM对网络进行处理，使得后续的客户端请求直接发给自己（Backup），而不是Primary，同时Backup的VMM不会再丢弃响应报文。这样Backup就接管成为了新的Primary。

> 如果Backup发生故障

如果Backup发生故障，Primary也要有相应的机制将Backup进行抛弃，停止向他发送Log Events，并表现为一个单点服务。

### Non-Deterministic Events 非确定性事件

对于客户端发送的报文，有些是可以直接执行并同步的，但并不是每条指令都可以直接由计算机内存内容确定下来。例如随机数、分布式ID、时间戳、多CPU的并发（**注意：VMware FT仅适用于单核的虚拟机**），虽然指令可能相同，但在不同机器直接执行产生的结果可能完全不同。这就是非确定性事件。

实际上由于所有的事件进行同步时，都是从Primary同步到Backup的，即要经过Log Channel。除了基本的数据包以外，可以在Log Event中增加一些信息，例如：

* 事件发生的指令序号：确保Primary和Backup接收指令的位置相同，这个指令是自机器启动以来所有执行的指令的相对序号。例如对于与时间戳相关的事件指令，其序号就是这条指令执行的序号，而可以在这条指令前加入一个伪造的模拟定时器中断的指令，这样Backup就知道在哪个指令位置进行事件的发生，并可以获取到这个数值
* Log Event的类型：是否涉及非确定事件

### Output Rule 输出控制

这个系统唯一的输出就是Primary对客户端请求的响应。

例如我们在虚拟机上跑一个Database server，客户端发送了一个对一个数据自增的请求，Primary应该对这个数+1，并返回新的数值。

假设一开始Primary和Backup都存了10，此时自增请求来到Primary，Primary正常自增并返回了11；然后这个请求发送给Backup，Backup将数值从10改为11，Backup的响应会被正常丢弃。

但如果Primary生成了回复给客户端后突然崩溃了，即还没有发送Log Event给Backup进行自增，此时就发生了数据不一致的问题。

论文中通过输出控制的方式进行解决：

在Primary给客户端发回响应之前，Primary的VMM会先发送这个Log Event给Backup，然后进行自己的指令执行；但这个指令的执行输出需要等到Backup完成指令后，返回一个ACK，Primary的VMM在收到ACK后才会进行输出，将响应发送回客户端。这就确保了Backup虚拟机一定也看到了这个请求，或者至少在Backup的VMM中缓存了这个请求。

可以看到，Primary必须等待Backup，而不可以异步。这对性能是较大的限制。

### Test And Set

我们始终假设Primary发生的是fail-stop故障，但实际上可能是因为网络问题，他们可以与其他客户端正常通信，但只是不能与对方通信。此时就会使得Primary和Backup都是在线的，从而导致脑裂（Split Brain）。论文中采用第三方**Test-And-Set服务来决定由谁上线**。

TestAndSet服务运行在非Primary和Backup的第三台机器上。这个服务会在内存保留一些标识位，当发送一个Test-And-Set请求时，它会设置一个标志位并返回旧值。Primary喝Backup都需要获取这个标志位（类似一个锁）。当请求到达时，不论是Primary收到了还是Backup收到了，都需要向这个服务进行一次请求。


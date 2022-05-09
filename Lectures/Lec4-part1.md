# Lec4 Fault-Tolerant Virtual Machines-PART 1⃣️

> 这一章主要是讨论容错（Fault-Tolerance）和复制（Replication）问题
>
> 主要参照论文 The Design of a Practical System for Fault-Tolerant Virtual Machines

## Replication

**复制（Replication）是解决容错最常用的机制。常用来解决fail-stop的问题。**

最简单的故障就是单台计算机的fail-stop。这是很容易可以通过复制来解决的。

> fail-stop是指，出现了故障后直接停止运行，而不是运算出错误的结果。

但是复制不能解决软件中的bug和硬件设计的bug。例如MapReduce的Master节点，如果我们采用复制的方法将其运行在两个节点，但Master程序中有一个bug，那么在两台master中都会得到错误的结果。

对于复制还有一些其他限制：两个副本，一个Primary和一个Secondary节点，我们经常假设两个副本的错误是相互独立的。但实际上，这种错误往往是有关联的，那么此时复制就毫无帮助，因为一个节点的错误会影响所有复制的副本。例如在一个计算机服务商的同一机房买机器，如果一台出现问题，其他可能也会有同样的问题缺陷；或者一个数据中心发生了物理损伤，如自然灾害，则多个“Replication”毫无意义。



## VMware Fault-Tolerance Replication

论文介绍了两种复制方式：State Transfer（状态转移），Replicated State Machine（复制状态机）

### 状态转移

如果服务器的两个副本始终保持同步，一旦Primary出现故障，由于Backup有所有的信息，则可以直接接管。这就是状态转移。即**Backup是Primary的完全拷贝**。当Primary故障时，Backup就可以从其保存的最新状态运行，所谓的状态转移就是将Primary的全部状态复制转移到Backup。**每过一段时间，Primary都会对自身内存做一份大的拷贝，通过网络同步到Backup。**

### 复制状态机

复制状态机不会在不同副本间发送状态，它只是**从Primary将这些客户端的操作指令/操作结果发送给Backup**。人们倾向于这种复制手段，因为并不需要做大副本的全部状态的同步，每次操作相对较小。但缺点是相对复杂。

## VMware Fault-Tolerance

VMware虚拟机，实际上是在实际的硬件上实现了一个虚拟机监控器VMM（Virtual machine Monitor），Monitor在同一硬件上模拟出多个虚拟的计算机。在VMM上可以运行一系列不同的OS，每一个都有自己的OS内核和应用程序。
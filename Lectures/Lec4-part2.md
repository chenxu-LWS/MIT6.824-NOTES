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

在VMware FT中，多副本没有用到虚拟机的本地磁盘，而是使用了Disk Server（一种远程磁盘）。即基本流程为客户端向Primary发送了一个请求，这个请求以网络数据包的形式发出。
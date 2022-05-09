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


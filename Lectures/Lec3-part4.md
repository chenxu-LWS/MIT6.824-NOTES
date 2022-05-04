# Lec2 Distributed File System - GFS - PART4⃣️

## GFS Read File

读请求需要哪些信息：文件名+offset

Master节点根据文件名在map中查找到ChunkID数组，根据offset计算获取ChunkID，之后Master找到存有这个Chunk的服务器列表并返回。

客户端从这些服务器中选择一个读取数据，在论文中这个选择是根据网络最近原则挑选（Google的数据中心的IP地址连续，因此根据IP判断网络位置远近），将读请求直接发到这个服务器。同时在客户端会**缓存Chunk与服务器的对应关系**。

接下来客户端会与选出的服务器通信，将ChunkID和偏移量发送给服务器。存储Chunk的服务器会在本地磁盘将每个Chunk存储为独立的Linux文件，通过普通的Linux文件系统进行管理。而这些文件是以ChunkID命名的。所以其要做的工作很简单，就是将这个文件找到并读取其中的数据，然后返回给客户端。



## GFS Write File

写请求需要哪些信息（这里假设只有**追加数据**的请求）：文件名+存储着数据的buffer

由于只需要追加数据到文件末尾，那么只需先向Master发起请求，获取最后一个Chunk的位置。即Master的最重要的任务就是告诉客户端应该与谁通信。

写文件时**只能向主Chunk进行写入**。在此要注意，对于某个特定的Chunk可能在某个时间点不存在制定的**主副本**，因此在写文件时需要考虑主副本不存在的情况：对于Master节点，如果Chunk的主副本不存在，Master会找到所有存有该Chunk**最新**副本的服务器。这里的最新是指服务器该Chunk的版本号与Master中该Chunk的版本号一致。这就是Chunk的版本号在Master节点上必须时非易失（Non-volatile）的原因。因为如果不进行持久化，重启后的Master不知道哪个Chunk版本是最新。

> 为什么不将所有保存该Chunk服务器的最大版本号作为最新版本？
>
> A：如果所有持有该Chunk的节点都正常返回了结果，那么这种方案是没问题的，但一旦有一些服务器发生故障或离线，此时Master可能只能获取到持有旧版本的Chunk服务器的响应。这时如果进行了写操作，则会发生不一致的情况。
>
> 但这种就会导致Master可能找不到存有最新版本的Chunk的服务器，那么就只有两种处理方式：1. Master一直等待 2. 返回错误给客户端，停止等待

之后，Master会增加版本号，并写入磁盘；Master从所有返回了最新版本号的服务器选出一个作为主Chunk，其他都是副Chunk，并发通知给他们Chunk的最新版本号。现在主Chunk就可以接收写请求了。而这个主Chunk并不是永恒的，而是有一个有效时间（论文中为60秒），60秒后必须停止作为主Chunk。这个机制确保不会同时有两个主Chunk。



如下图所示，Master节点告诉了客户端哪台机器为主Chunk，哪些机器为副Chunk。客户端会将要追加的数据发送给Primary Replica（主Chunk）和Secondary Replica（副Chunk），这些服务器会将数据写入到一个临时位置。也就是说这些数据并不会直接追加到文件中。当所有服务器都返回确认消息，客户端会向Primary机器发送消息说“你和Secondary机器都有了要追加的数据，请进行追加”。Primary可能会收到大量的并发请求，其会以某种顺序一次只执行一个请求。Primary会查看这个Chunk，确保有足够空间追加，然后将追加的数据写在Chunk末尾。再通知所有的Secondary服务器进行追加。

但对于Secondary机器来讲，可能成功或失败，例如磁盘空间不足或故障等等。如果Secondary真正将数据存储到了Chunk，会回复"yes"给Primary。如果Primary收到了所有来自Secondary的"yes"，则会向客户端返回写入成功。如果至少一个Secondary机器没有回复"yes"，则返回写入失败。然后客户端应该再次发起整个过程进行追加。

也就是说这种部分成功部分失败的情况是可以被接受的，因为记录了Chunk的版本号。下次再进行追加时直接从当前这个部分成功的继续动作即可。

<img src="/Users/liuwenshuo/Documents/Notes/6.824/Lectures/image-20220504222337300.png" alt="image-20220504222337300" style="zoom:50%;" />
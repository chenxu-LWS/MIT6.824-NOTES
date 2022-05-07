# Lec3 Distributed File System - GFS - PART4⃣️

## GFS一致性

GFS并不保证副本的一致性，举例如下：

假设我们在对一个Chunk追加数据，这个Chunk有三个副本（含一个Primary，两个Secondary）。

当客户端发送一次追加数据的请求后，将数据A加到Chunk末尾。假设这次都成功了，那么三个副本的数据变成如图所示：

<img src="/Users/liuwenshuo/Documents/Notes/6.824/Lectures/image-20220507101817017.png" alt="image-20220507101817017" style="zoom: 33%;" />

第二次追加数据B，但由于网络问题，消息只被两个副本收到。

<img src="/Users/liuwenshuo/Documents/Notes/6.824/Lectures/image-20220507101849720.png" alt="image-20220507101849720" style="zoom: 33%;" />

第三次追加数据C，且这个客户端的主Chunk仍为原来的Primary，Primary选择了偏移量并将偏移量告诉了Secondary。

<img src="/Users/liuwenshuo/Documents/Notes/6.824/Lectures/image-20220507102054983.png" alt="image-20220507102054983" style="zoom: 33%;" />

对于数据B，由于Secondary2写入失败，会返回给客户端失败的消息，因此会重新进行B写入。

<img src="/Users/liuwenshuo/Documents/Notes/6.824/Lectures/image-20220507104842244.png" alt="image-20220507104842244" style="zoom: 33%;" />

之后如果客户端读文件，读到的内容取决于读取的是Chunk的哪个副本，可以看到副本之间并不能保证一致性。

GFS这样设计的理由是足够简单，但同时也带来了一些一致性问题和数据错乱问题。应用程序需要接收数据的乱序。

实际上如果想将GFS变成一个强一致的系统，需要进行全新的设计，其中有以下几点需要考虑：

* **重复请求问题的解决**：Primary需要探测重复请求，当第二个写入数据B的请求到达时，Primary需要知道这个数据已经执行了，从而防止数据出现两次。
* **重试或移除**：对于Secondary，如果Primary要求Secondary进行一个操作，Secondary必须保证执行，而非只是返回错误给Primary。因此Secondary需要一些机制对Secondary应该进行的操作进行重试或有一种机制将错误的Secondary从系统中移除。
* **隔离与两阶段提交**：当Primary要求Secondary追加数据时，直到Primary确保所有的Secondary都执行了数据追加之前，Secondary必须小心不能将数据暴露给读请求。在第一个阶段，Primary向Secondary发请求；第二个阶段如果所有Secondary都回复了yes，这时Primary才完成。这种方式称为两阶段提交（Two-phase commit）
* **新Primary**：当一个Primary崩溃了，一个Secondary会接替成为新的Primary，但此时新Primary与其他Secondary的操作可能有不同，因为部分副本没有收到旧Primary崩溃前发出的请求。所以新Primary上线时，需要显式的进行与Secondary的同步，以确保操作历史的末尾相同。

## GFS的局限性

GFS最严重的局限性在于其只有一个Master节点真正工作。

* Master存储的表单越来越大，总有一天会消耗殆尽
* 单个Master节点要承载几千个客户端的请求，而Master的CPU每秒只能处理数百个请求，再加上Master还需要将部分数据写入磁盘，更是扛不住如此大量的并发数
* GFS各副本之间数据并不保证一致性，甚至会产生不可预期的奇怪数据
* Master的切换并不是自动化的，GFS需要人工干预处理永久故障的Master并更换新的服务器
# LAB1-MapReduce



## TARGET

实现一个分布式的MapReduce。包含两个程序，一个调度程序（Coordinator）和工作程序（Worker）。调度程序只有一个进程，而worker有多个进程并行。在实际的系统中，worker程序会跑在不同的机器上，但在这个lab中会跑在单个机器上。

worker通过RPC与coordinator通信。每个worker进程都向coordinator要task，从一个或多个文件中读入task的输入，执行task，将task的输出写入一个或多个files中。coordinator应该知晓worker有没有在限定时间内（在lab中是10秒）完成任务，如果没有则需要将这个任务重新分配给另一个worker。

## GET STARTED

coordinator和worker的主要功能在main/mrcoordinator.go和main/mrworker.go中。但不要修改这两个文件，我们应该在mr/coordinator.go和mr/worker.go以及mr/rpc.go中实现我们的功能。

如何验证我们的实现？

将pg-*.txt作为参数传入并运行mrcoordinator.go，每个文件对应一个分区（split）作为map任务的输入。

```shell
$ go run -race mrcoordinator.go pg-*.txt
```

将编译好的插件wc.so（word count，wc.go中实现了map函数和reduce函数，作为plugin的形式介入mrworker）作为参数传入并运行**多个**mrworker.go

```shell
$ go run -race mrworker.go wc.so
```

当worker和coordinator完成后，会将wordcount的结果输出道mr-out-*中。

同时实验还提供了main/test-mr.sh用于测试我们的coordinator和worker是否正确。这个test脚本会检查：

* 你实现的map和reduce tasks是否并行运行；
* 检查当一个worker出现故障时，你的coordinator能否解决；
* 是否对于每个reduce任务都产出了一个mr-out-X文件

当你完成后，输出将是这样的：

```shell
$ bash test-mr.sh
*** Starting wc test.
--- wc test: PASS
*** Starting indexer test.
--- indexer test: PASS
*** Starting map parallelism test.
--- map parallelism test: PASS
*** Starting reduce parallelism test.
--- reduce parallelism test: PASS
*** Starting crash test.
--- crash test: PASS
*** PASSED ALL TESTS
```

将coordinator注册为一个RPC server，检查其所有的方法是否都适用于RPC（有三个输入）；注意其中*DONE*方法不是通过RPC调用的。

## RULES

1. map阶段会将中间值keys分隔到*nReduce*个buckets，每个bucket对应一个reduce task。（*nReduce*是由mrCoordinator.go传递到MakeCoordinator方法的一个参数）。因此每个mapper应该创建*nReduce*个中间文件，这些中间文件后续会被reduce tasks使用。
2. 输出到文件的文件名：worker应该将第X次reduce任务的结果，输出到mr-out-X文件中。
3. 输出到文件的格式：一个mr-out-X文件应该包含reduce函数输出的k-v，放到一行。使用golang的"%v %v"形式进行输出。
4. 你可以修改mr/worker.go`、`mr/coordinator.go`和`mr/rpc.go。
5. worker应该将map tasks的中间输出也存储为文件，这样reduce tasks才能读取
6. main/mrcoordinator.go需要mr/coordinator.go实现一个Done()函数，当整个mapreduce任务结束后返回true。这样mrcoordinator才能结束。
7. 当任务结束时，worker进程应该退出。最简单的实现方式是用call()函数的返回值实现：当worker无法与coordinator通信时，worker可以假设coordinator已经退出了，所有的任务都已经完成了，从而worker自己也可以退出了。或者根据你的设计，可能会用coordinator给worker发送一个"please exit"的假的task，让worker退出，这样也是有效的。

## HINTS

1. 你可以从修改mr/worker.go的Worker方法开始做起，让它发送一个RPC调用给coordinator去请求一个任务。然后修改coordinator，去响应这个请求，响应内容可以是一个还未进行map task的文件的文件名。然后修改worker去读取这个文件，调用Map方法（参考mrsequential.go）。
2. 如果你修改了mr/下的任何文件，你需要重新build你的map、reduce函数的实现

```shell
go build -race -buildmode=plugin ../mrapps/wc.go
```

3. 这个lab依赖于我们所有的worker进程都共享一个文件系统。这在所有的worker进程都跑在同一台机器上时是非常简单直接的。但如果跑在多台机器上，则需要一个像GFS这种分布式的文件系统。

4. 如何命名中间文件？可以尝试将中间文件命名为*mr-X-Y*，其中X是map task的编号，Y是reduce task的编号

5. map task需要有一种方法存储中间生成的k-v键值对到文件中，且需要可以从文件中读取出来。一个简单的方法是使用golang的encoding/json包，将这些k-v对存储为JSON格式。

   存储：

```go
enc := ejson.NewEncoder(file)
for _, kv := ... {
  err := enc.Encode(&kv)
```

​		读取：

```go
dec := json.NewDecoder(file)
for {
  var kv KeyValue
  if err := dec.Decode(&kv); err != nil {
    break
  }
  kva = append(kva, kv)
}
```

6. worker的map task的部分，可以使用worker.go中的ihash(key)这个函数来选择一个给定key的reduce task。

7. 你可以参考mrsequential.go中的一些代码，用于读取Map的输入文件、对Map和Reduce的中间k-v进行排序、将Reduce的输出存储在文件中

8. coordinator作为一个RPC server是并发的，注意对一些共享数据的保护

9. 通过-race参数使用golang的race detector

10. worker有些时间需要等待，例如reduce任务必须等待其之前的map任务完成后才能进行。一个较好的解决方式是每隔一段时间就向coordinator请求task，使用time.Sleep函数控制间隔。另一个方式是在coordinator中使用RPC handler去等待，使用time.Sleep或者sync.Cond。golang会在其线程中对每个RPC调用创建一个handler，所以一个handler的等待不会阻塞其他RPC的处理。

11. coordinator无法区分以下几种worker

    * 已经发生了故障的worker
    * 存活但由于某种原因停止了的worker
    * 正在处理task但太慢导致无法使用的worker

    我们可以做的最好的就是让coordinator等待一段时间，如果没有返回，则放弃并重新发布给其他的worker。

12. 如果你选择实现备份任务（Backup Tasks），
13. 如果要测试异常的恢复，你可以使用mrapps/crash.go作为mapreduce的plugin。它会在map和reduce过程随机退出。
14. 为了确保出现异常时，没有其他进程感知到部分写入的文件，mapreduce提到了使用临时文件，并在完全写入结束后再进行重命名的方法。你可以使用ioutil.TempFile创建一个临时文件并用os.Rename进行重命名。
15. test-mr.sh在其子目录的mr-tmp中启动进程，所以如果出现问题，进入这个目录即可查看中间文件或输出文件。你可以暂时修改test-mr.sh，在出现失败测试时退出当前脚本，不再进行新的测试，这样脚本才不会重写输出文件。
16. test-mr-many.sh是一个给test-mr.sh加了超时参数的测试脚本。实际上就是跑了一次test-mr.sh。超时时间作为参数传递。注意我们不可以同时跑多个test-mr.sh，因为coordinator会重复使用同一个socket，导致冲突异常
17. golang的RPC只能发送首字母大写的struct。sub-struct也必须首字母大写！
18. 当调用方向一个RPC系统传递一个指向期望得到的响应结构的指针，这个指针指向的对象必须是未分配空间（zero-allocated）的，不要在call前设置任何字段。如果不遵守这个要求，当你执行RPC时，这个字段的写入不会生效；在调用方，这个被修改的值会永远保存，不会变成真正的返回值。因此应该这样写：

```go
args := someArgs{}
args.X = 1
reply := SomeType{} // 不要修改其中的任何属性
call(..., &args, &reply)
```
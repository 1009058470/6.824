[lab-mr](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)
# 介绍
在这个实验室中，您将构建一个MapReduce系统。您将实现一个工作进程，它调用应用程序映射和Reduce函数，并处理读写文件，以及一个主进程，它将任务分发给工作人员并处理失败的工作人员。您将构建类似于MapReduce paper的内容。

# 合作政策

你必须写完6.824的所有代码，除了作为作业一部分的代码。你不允许看其他人的解决方案，你也不允许看前几年的解决方案。你可以和其他同学讨论作业，但你不能互相看或抄袭别人的代码。这个规则的原因是，我们相信您将通过自己设计和实现您的实验室解决方案学到最多的东西。


请不要发布您的代码，也不要将它提供给当前或将来的6.824学生。默认情况下，github.com存储库是公共的，所以请不要将代码放在那里，除非您将存储库设置为私有。您可能会发现使用MIT的GitHub很方便，但是一定要创建一个私有存储库。

# 软件


您将在Go中实现这个实验室(以及所有的实验室)。Go网站包含很多教程信息。我们将使用Go版本1.13给你的实验室打分;你也应该使用1.13。你可以通过运行Go版本来检查你的Go版本。


我们建议您在自己的机器上进行实验，这样您就可以使用您已经熟悉的工具、文本编辑器等。或者，你也可以在实验室工作。
# 开始
获取6.824实验软件:
``` shell
git clone git://g.csail.mit.edu/6.824-golabs-2020 6.824

```
我们在src/main/mrsequentii.go中为您提供了一个简单的顺序mapreduce实现。它运行映射，并在单个进程中每次减少一个映射。我们还为您提供了几个MapReduce应用程序:mrapps/wc中的单词计数。在mrapps/indexer.go中设置一个文本索引器。你可以运行顺序如下:

``` shell
cd 6.824
# export GOPATH=$GOPATH:$PWD not use GOPATH
cd src/main
go build -buildmode=plugin ../mrapps/wc.go
go run mrsequential.go wc.so pg*.txt
more mr-out-0
```

mrsequential.go将其输出留在文件mr-out-0中。输入来自名为pg-xxx.txt的文本文件。
您可以随意从mrsequential.go借用代码。你也应该看看mrapps/wc。去看看MapReduce应用程序代码是什么样的。
# 工作

您的工作是实现一个分布式MapReduce，它由两个程序组成，主程序和辅助程序。将只有一个主进程和一个或多个并行执行的辅助进程。在一个真实的系统中，工作人员会在很多不同的机器上运行，但是在这个实验室中，你会在一台机器上运行所有的机器。工作人员将通过RPC与主进程对话。每个工作进程将向主进程请求一个任务，从一个或多个文件中读取任务的输入，执行任务，并将任务的输出写入一个或多个文件。如果一个工人没有在合理的时间内完成任务(对于这个实验室，使用10秒)，主人应该注意到，并将相同的任务交给另一个工人。


我们已经给出了一些代码来启动你。主程序和辅助程序的“主”例程位于main/mrmaster.go,main/ mrworker.go;不要更改这些文件。你应该把你的实现放在mr/master.go, mr/worker.go,mr/rpc.go。


下面介绍如何在word-count MapReduce应用程序上运行代码。首先，确保word-count插件是新建的:
```shell
go build -buildmode=plugin ../mrapps/wc.go
```

在main目录中，运行master。
```shell
rm mr-out*
go run mrmaster.go pg-*.txt
```
go run mrmaster.go的pg-*.txt参数是输入文件,每个文件对应一个“split”，并且是一个Map任务的输入。


在一个或多个其他窗口，运行一些workers:
```shell
go run mrworker.go wc.so
```

master,workers完成后，看看输出输出。完成实验后，输出文件的排序并集应该与顺序输出相匹配，如下所示:
```shell
cat mr-out-* | sort | more
```

我们为您提供了一个main/test-mr.sh的测试脚本。测试检查wc和indexer MapReduce应用程序在给定pg-xxx.txt文件作为输入时是否产生正确的输出。测试还会检查您的实现是否并行运行映射和Reduce任务，以及您的实现是否从运行任务时崩溃的workers中恢复。
如果你现在运行测试脚本，它会挂起，因为主程序永远不会完成:
```shell
cd src/main
sh test-mr.sh
```
可以在mr/master.go的Done函数中将ret:= false改为true。master exits:
```shell
sh ./test-mr.sh
*** Starting wc test.
sort: No such file or directory
cmp: EOF on mr-wc-all
--- wc output is not the same as mr-correct-wc.txt
--- wc test: FAIL
```

测试脚本期望在名为mr-out-X的文件中看到输出，每个reduce任务对应一个文件。mr/master.go,mr/worker.go的空实现不生成这些文件(或者做很多其他事情)，因此测试失败。
当您完成时，测试脚本输出应该如下所示:
```shell
sh ./test-mr.sh
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

您还将看到来自Go RPC包的一些错误，看起来像这样:
```shell
2019/12/16 13:27:09 rpc.Register: method "Done" has 1 input parameters; needs exactly three
```

忽略这些消息。

一些规则: 
- map阶段应该将中间键划分为用于nReduce reduce任务的桶，其中nReduce是main/mrmaster.go的参数传递给MakeMaster()。
- work实现应该将第X个reduce任务的输出放在文件mr-out-X中。
- 一个mr-out-X文件应该包含每个Reduce函数输出的一行。这一行应该使用Go“%v %v”格式生成，并使用键和值进行调用。看一下main/mrsequential.go,转到注释为“this is the correct format”的行。如果您的实现太偏离这种格式，测试脚本将会失败。
- 你可以修改mr/worker.go, mr/master.go, mr/rpc.go。您可以临时修改其他文件以进行测试，但要确保代码与原始版本兼容;我们将使用原始版本进行测试。
- worker应该将中间Map输出放在当前目录的文件中，worker可以将其作为输入读取Reduce任务。
- main/mrmaster.go希望mr/master.go实现Done()方法.该方法在MapReduce作业完全完成时返回true;在这一点，mrmaster.go会退出。
- 当job完全完成时，workers应该退出.实现这一点的一种简单方法是使用call()的返回值,如果worker没有联系主进程，它可以假设主进程已经退出，因为作业已经完成，所以worker也可以终止。根据您的设计，您可能还会发现有一个“请退出”伪任务是很有帮助的，master可以将这个伪任务交给worker。

提示:
- 开始的一种方法是修改mr/worker.go的Worker()，以便向master发送请求任务的RPC。然后修改master，以响应一个尚未启动的映射任务的文件名。然后修改worker以读取该文件并调用application Map函数，如mrsequential.go中所示。

- 应用程序映射和Reduce函数在运行时使用Go plugin包从名称以.so结尾的文件加载。

- 如果你改变了mr/目录中的任何东西，你可能需要重新构建你使用的MapReduce插件，比如go build -buildmode=plugin ../mrapps/wc.go

- 这个实验室依赖于共享文件系统的workers。当所有workers都在同一台机器上运行时，这是很简单的，但是如果workers在不同的机器上运行，则需要像GFS这样的全局文件系统。

- 中间文件的合理命名约定是mr-X-Y，其中X是映射任务编号，Y是reduce任务编号。

- worker的map任务代码需要一种方法来将中间键/值对存储在文件中，这种方法可以在reduce任务期间正确地读回。一种可能是使用Go的编码/json包。要将键/值对写入JSON文件:
    ```golang
    enc := json.NewEncoder(file)
    for _, kv := ... {
        err := enc.Encode(&kv)
    ```
    并读取这样的文件回来：
    ```golang
    dec := json.NewDecoder(file)
    for {
        var kv KeyValue
        if err := dec.Decode(&kv); err != nil {
            break
        }
        kva = append(kva, kv)
    }
    ```

- work的map部分使用worker.go中的ihash(key)函数对于给定的key选择reduce任务

- 您可以从mrsequential.go中窃取一些代码，用于读取Map输入文件、排序Map和Reduce之间的中间键/值对，以及将Reduce输出存储在文件中。

- master作为RPC服务，是并发的。不要忘记用锁处理共享数据。

- 使用Go的race检测器，使用`go build -race` 和 `go run -race`。`test-mr.sh`有一条解释展示了如何在测试中启用race检测器

- workers有时会需要等待。例如，reduce人物不能开始直到最后一个map任务完成。一种可能是workers定期向master请求任务，请求间隔中使用`time.Sleep()`休眠.另一种可能是master相关的RPC处理程序有一个等待的循环，可以是`time.Sleep()`或者`sync.cond`.Go在自己的线程中为每一个RPC运行处理程序，所以一个处理程序等待不会阻止master运行其他PRCs。

- master无法可靠地分辨崩溃的workers，存活但因一些原因停滞的wokers和运行中但处理速度很慢导致无法使用的workers。最好的做法是master等待一些时间，然后放弃，把任务重新分配给另一个worker。在这个lab中，master等待十秒钟后可以假设worker已经崩溃（当然，可能没有崩溃）。

- 要测试崩溃恢复，可以使用mrapps/crash.go应用程序插件，它随机地存在Map和Reduce函数中。

- 为了确保没有在出现崩溃的情况下观察到部分写入的文件，MapReduce paper提到了使用临时文件并在写入完成后对其进行原子重命名的技巧。可以使用ioutil.TempFile创建临时文件，使用os.Rename以原子方式重命名。

- test-mr.sh运行子目录mr-tmp中的所有进程，如果除了问题，你想查看中间文件或者输出文件，请查看那里。

# 完成

 运行 test-mr.sh，并通过所有tests
---
title: Hbase性能压测工具
date: 2018-11-10 12:40:25
tags:
        - Hbase
        - Java
---

# PerformanceEvaluation源码阅读

近日在工作需要对集群的hbase做些简单的测试，所以接触了PerformanceEvaluation这个类，作为HBase自带的性能测试工具，其主要就是模拟多用户同时访问集群进行压测。

## 用法 & 主要参数
```
bin/hbase pe <options> <command> <nclients>
```
倒数第二个参数command为hbase提供的几种测试用例，最后一个参数为线程数

测试用例常用的有：
- sequentialRead ———— 顺序读测试
- sequentialWrite ———— 顺序写测试
- randomRead ———— 随机读测试
- randomWrite ———— 随机写测试
- scan ———— 扫描 (读每一行)
- scanRange10/100/1000 ———— 指定范围内随机扫描，返回10/100/1000个数据

主要参数有：
- nomapred —— 使用MapReduce的方式启动多线程测试还是启动本地多线程的方式，通常使用本地多线程方式。如果没有安装MapReduce加上此参数表示不是mr
- rows —— 指定**每个线程**处理的数据行数，总共测试行数等于线程数*rows
- size —— 总测试的数据大小，单位为GB。 这个参数与rows**互斥**，不要两个参数一起设。在使用RandomRead和RandomSeekScan测试时，这个size可以用来指定读取的数据范围。这个值在Read时非常重要，如果设的不好，会产生很多返回值为空的读，影响测试结果 
- valueSize —— 写入HBase的value的size，单位是Byte，默认1024字节
- oneCon —— 多线程运行测试时，所有线程使用一个连接还是每个线程一个连接。默认值为false，每个thread都会启一个Connection，建议把这个参数设为True *(涉及底层netty的线程模型，hbase2.0后可指定连接数)*
- presplit ——  表的预分裂region个数
- inmemory —— 会将数据尽量放在内存中，默认是false，为了保证测试准确性，建议保持为false
- table —— 测试表的名字，默认为TestTable
- compress —— 指定压缩方式，默认是NONE
- filterAll —— 加上此参数，则server端scan出来的结果不再返回给client端，用于单纯测试server端的性能
- autoFlush —— 默认为false，即PE在写测试时用的是BufferedMutator，BufferedMutator会把数据攒在内存里，达到一定的大小再向服务器发送，如果想明确测单行Put的写入性能，建议设置为true。autoFlush为false会影响统计的准确性，因为在没有攒够足够的数据时，put操作会立马返回，根本没产生RPC，但是相应的时间和次数也会被统计在最终结果里 (源码中好像没有看到这个参数的作用？？)

## 源码解读
1. 继承了Configured类，表示已被配置了Configuration文件，在PE的构造函数中调用了super(conf)，将配置文件保存了起来

2. 在run方法中，对参数进行处理。这种处理参数的方法以后可以借鉴。将所有参数以空格分割，转成linkedList，再在一个方法中解析该list，每次poll一个参数进行判断赋值给TestOptions类保存参数。
并且通过这种方法控制命令行格式，若判断到一个参数为测试用例的其中一种，则下一个参数必须为线程数，最后在判断list是否为空。
>从parseOpts方法中能看到，size参数和rows参数必须设置一个且只能设其一。如果指定size，这算出要测试的总行数再除以线程数，得到每个线程要操作的rowkey范围；若指定每个线程处理的行数，则同理求出size大小

> TestOptions保存这全部的参数信息，并且提供了一个方法可以从一个TestOptions对象复制出另一个该对象的方法，供多线程测试时，每个线程维持一份不同的参数

3. 通过反射构造出测试用例的类类型Class，所有测试用例类都通过TableTest继承Test，所以是Test的子类

4. runTest方法中，搜先是调用了checkTable方法来判断*测试表是否需要重建*，主要判断逻辑：
	- 在只读测试中，必须指定一张存在的表
	- 如果测试表存在，但与指定的region数不同，则要重建
	- 如果是写测试，测试表的分区策略或副本数与指定参数不同，则也要重建
目的就是为了提供一个与指定参数完全符合的测试环境，主要就是region数，分区策略和副本数要一致，这样测试结果才准确。 最后根据nomapred决定是本地多线程测试还是启动mapreduce测试。

5. doLocalClients的主要逻辑是创建和线程数相同的Future对象用于取回测试结果，再创建一个线程池向提交n个线程。为每个线程复制一个TestOptions对象，以维持每个线程不同的参数，例如startRow。 再调用runOneClient方法

6. runOneClient是每个线程单独处理的方法，通过传入的参数反射构造出Test测试用例类，再调用Test的test方法(这里使用了模板方法的设计模式)。最后返回RunResult对象，保存着这个线程的一些性能指标，返回出去可用于统计平均值和方法等总体性能指标。

## 注意的问题
1. 在进行读测试之前要准备数据，建议使用SequentialWrite测试用例。
>在SequentialWrite中，PE会给每个线程设置偏移量，保证0 ~ 9999这10000个行（会把所有数字扩展成26位等长的byte数组）一行不差地写入HBase。如果是RandomWriteTest，在每个线程中会随机生成一个0 ~ 9999之前的数字写入（--row=1000代表每个线程会写1000次）。由于是随机，会造成中间有些行没有写入，那么在读取测试时，读到的就是空行，影响测试结果。

2. 建议使用--size参数而不是--rows
>size参数指定后，具体执行多少行PE内部会自己去算。假设我这里填的是--row=1000，线程数是10，那么写入的数据范围是0~9999。当我在做RandomReadTest时，如果需要修改线程数，比如我想测20个线程并行读，那么数据读取的范围将是0 ~ (1000*(20-1))，很大一部分读是空读！你当然可以根据线程数来调整读测试时row变量的值，使读的整体范围不超过写入的数据范围，但是row的大小影响了整体测试的时间，而统一用size你就啥都不用管了。

3. 
> 在读测试时不要加关于表的任何参数，如presplit，如果加了PE会将表重建。valueSize和size的值要与准备数据时写测试用例的参数保持一致，PE靠这两个值来算数据的范围和行数。

## 内部类解读
Test是抽象类，提供了很多hook，类似模板方法供子类实现
比如：TableTest实现了Test的onStartup和onTakedown方法，控制每个测试用例Table对象的创建(Table对象的创建比较消耗hbase资源)；所有测试用例类都实现了testRow类方法，实现各自测试方法对每行数据的处理。**所以从Test类基本能看到整个测试的流程**

整个test方法调用链有三条：
1. test ——>  testSetup:负责线程和连接的关系 ——>  onStartup: 通过线程各自的连接得到table对象，由TableTest或BufferedMutatorTest实现
2. testTimed: 得到起止的rowkey  ——>  testRow：由每个测试用例类实现各自逻辑
3. testTakedonw  ——>  打印性能指标，onTakedown：关闭table对象

其中RandomWriteTest和SequentialWriteTest这两个写测试用例继承BufferedMutatorTest,内部是一个BufferedMutator，用于异步批量写数据，实现大致和TableTest一样，只是把Tabel接口换成了BufferedMutator接口

Imcrement、Append和CheckAnd开头的测试类，有类似CAS操作，则继承了CASTest这个基类

##参考资料
- [HBase2.0中的Benchmark工具 — PerformanceEvaluation](https://yq.aliyun.com/articles/594384)
- [HBase PerformanceEvaluation机制分析](https://blog.csdn.net/bryce123phy/article/details/77905538)
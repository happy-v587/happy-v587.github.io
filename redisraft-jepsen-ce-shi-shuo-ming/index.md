# RedisRaft - 混沌测试说明



- redis-raft 仓库提供了基于 docker 的运行方式
![image](/images/redisraft-jepsen-ce-shi-shuo-ming.md/img-01.png)
- 运行的代码存储在 https://github.com/redislabs/jepsen-redisraft

---

### 关于 jepsen-redisraft 仓库的说明

<img width="359" alt="image" src="/images/redisraft-jepsen-ce-shi-shuo-ming.md/img-02.png">

0、程序入口

    在 project.clj 中定义了 `:main jepsen.redis.core`
    然后打开 `src/jepsen.readis/core.clj` 文件
    找到文件最后 `defn -main` 即为入口

1、【编译】如何编译 redis、redis-raft 

    在 `defn -main` 中调用了 `defn redis-test`
    在 `defn redis-test` 中调用了 `rdb/redis-raft`
    在 `defn redis-raft` 中实现了 jepsen 要求的
        - `db/DB` 接口：就两个函数 `setup! [db test node]` 和 `teardown! [db test node]`
            - 在 `setup! [db test node]` 中执行了 `install-build-tools!` 、`build-redis!`、`build-redis-raft!`。具体实现类似 bash 逻辑 `mkdir -p xx; git clone xxx; make`

2、【启动集群】如何启动 raft cluster 集群

    还是在 `setup! [db test node]` 中，执行了cmd
        - `redis raft.cluster init` -> `cli! :raft.cluster :init`
        - `redis raft.cluster join` -> `(cli! :raft.cluster :join (str (jepsen/primary test) ":6379")))) `

3、【验证数据】如何生成数据、如何写数据、如何验证数据

    回到 `defn redis-test` 处
    在这里初始化了 workload `let [workload ((workloads (:workload opts)) opts)]`
    而workload的实现在 `src/jepsen.readis/append.clj` 文件的 `defn workload` 函数
    这个函数具体逻辑直接调用的 `jepsen.tests.cycle.append` 测试方法，这个方法包含了数据的生成、校验。
        - 生成逻辑底层其实就是 `elle.list-append.gen`
        - 校验逻辑底层也是 `elle.list-append.check` 的实现

    数据写入是通过定义一个`defrecord Client [conn]`，然后实现`jepsen.client/Client`协议。接口包括：open! close! setup! invoke! teardown!
        - 因为我们要连接的是 redis，所以底层使用的 client 是 https://github.com/taoensso/carmine（类似 go-redis 的存在）
        - 主要逻辑是实现 invoke! ，被读写的数据通过参数传来，所以 invoke! 只负责处理读写操作的执行。在底层就是 `lrange` 和 `rpush`。 最后将读写后得到的数据返回
        - 返回的数据，jepsen会记录到history文件中，最终会通过分析history文件判断数据是否有问题

    关于数据校验可以看 https://github.com/jepsen-io/elle

4、【注入故障】如何注入故障

    jepsen 支持自定义故障，对应接口 `jepsen.nemesis/Nemesis`
        - 比如： 在 `src/jepsen.redis/nemesis.clj` 文件中，`defn member-nemesis` 函数就是 `Nemesis` 接口的一个实现，这个实现的故障是一个节点的 `join` 或者 `leave`
    
    故障的注入也是通过生成器的方式，具体的生成器逻辑可以自己定义。如口在 `src/jepsen.redis/nemesis.clj` 文件的 `defn package` 函数
        - 分为 island-generator 和 mystery-generator
            - island-generator：隔离（网络隔离？还是member leave？通过哪个实现？）
            - mystery-generator：自定义（kill all、kill some、kill part、start、leave、join）

5、【测试框架】put it together

https://blog.csdn.net/FL63Zv9Zou86950w/article/details/117457782

![image](/images/redisraft-jepsen-ce-shi-shuo-ming.md/img-03.png)
  
以分布式数据库集群为例，Jepsen的工作大致包含以下几步：

      Jepsen在控制节点(Control Node)上作为Clojure程序运行；
      部署需要测试的分布式数据库集群(Distributed System)，确认其工作正常；
      控制节点启动进程，作为分布式数据库节点(DB Node)的SSH客户端；
      生成器(Generator)进程为每个SSH客户端生成具体要执行的读写操作；
      这些SSH客户端将在分布式数据库中执行这些读写操作；
      每个操作从启动到结束的过程都会记录下来；
      当读写操作进行时，克星进程(Nemesis)对分布式数据库进行故障注入，同样也是由生成器来调度和管理；
      测试完成，分布式数据库集群会被销毁。Jepsen使用检查器(Checker)进程对测试过程的历史记录进行分析，并生成最终的图表和报告。

---

### 语法、框架学习

- clojure 语法：https://www.w3cschool.cn/clojure/clojure-j5w81wf2.html
- jepsen 文档：https://jepsen-io.github.io/jepsen/jepsen.cli.html#var-test-all-cmd
- jepsen 训练课：https://jaydenwen123.gitbook.io/zh_jepsen_doc

### 数据验证相关

elle 数据验证工具：https://github.com/jepsen-io/elle
> Elle is a transactional consistency checker for black-box databases.
> As a user, your main entry points into Elle will be elle.list-append/check and elle.rw-register/check.
> Elle has a broad variety of anomalies and consistency models; see elle.consistency-model for their definitions. Not every anomaly is detectable, but we aim for completeness.

elle-cli 支持更多业务模型：https://github.com/ligurio/elle-cli
> is a command-line frontend to transactional consistency checkers for black-box databases. In comparison to Jepsen library it is standalone and language-agnostic tool.

history 文件格式（jepsen使用）: https://github.com/jepsen-io/history
> To analyze these systems, Jepsen uses a history: a totally ordered log of concurrent logical operations.

EDN 文件格式（elle使用）： https://github.com/edn-format/edn
> Extensible Data Notation

JSON -> EDN 文件格式转换：https://github.com/borkdude/jet
> CLI to transform between JSON, EDN, YAML and Transit using Clojure.

### 其他

启动交互式命令行

    # REPL 的环境，即 Read-Eval-Print Loop —- “读取-求值-输出” 循环
    ~/code/cpp/redisraft/jepsen/docker/control/lein repl

调用 list-append/gen 接口

    jepsen.redis.core=> (ns jepsen.redis.core (:require [jepsen.tests.cycle.append :as append]))
    nil
    jepsen.redis.core=> (append/gen)
    (gen/seqone ({:type :invoke, :f :txn, :value [[:append 2 1] [:r 1 nil]]} {:type :invoke, :f :txn, :value [[:append 2 2] [:r 1 nil]]} {:type :invoke, :f :txn, :value [[:r 1 nil] [:append 2 3]]} {:type :invoke, :f :txn, :value [[:append 2 4]]} {:type :invoke, :f :txn, :value [[:append 0 1]]} {:type :invoke, :f :txn, :value [[:r 1 nil]]} {:type :invoke, :f :txn, :value [[:append 2 5]]} {:type :invoke, :f :txn, :value [[:append 2 6] [:r 0 nil]]}))
    
将 gen 结果格式化展示

    {:type :invoke, :f :txn, :value [[:append 2 1] [:r 1 nil]]} 
    {:type :invoke, :f :txn, :value [[:append 2 2] [:r 1 nil]]} 
    {:type :invoke, :f :txn, :value [[:r 1 nil] [:append 2 3]]} 
    {:type :invoke, :f :txn, :value [[:append 2 4]]} 
    {:type :invoke, :f :txn, :value [[:append 0 1]]} 
    {:type :invoke, :f :txn, :value [[:r 1 nil]]} 
    {:type :invoke, :f :txn, :value [[:append 2 5]]} 
    {:type :invoke, :f :txn, :value [[:append 2 6] [:r 0 nil]]}



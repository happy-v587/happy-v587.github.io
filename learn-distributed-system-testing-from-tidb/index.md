# 从 TiDB 学习分布式数据库测试


## 前言

最近在研究数据库正确性测试相关的内容，恰好看到TiDB数据库在这方面的工作，很受启发，故写此文章。

推荐下一些TiDB官方好文章：

- 《分布式系统测试那些事儿 – 理念》https://cn.pingcap.com/blog/distributed-system-test-1/
- 《分布式系统测试那些事儿 – 错误注入》https://cn.pingcap.com/blog/distributed-system-test-2/
- 《分布式系统测试那些事儿 – 信心的毁灭与重建》https://cn.pingcap.com/blog/distributed-system-test-3/
- 《TiDB 混沌工程实践：如何打造健壮的分布式系统？》https://cn.pingcap.com/blog/tidb-chaos-engineering/
- 《基于 Chaos Mesh 和 Argo 打造分布式测试平台》https://cn.pingcap.com/blog/building-a-distributed-test-platform-based-on-chaos-mesh-and-argo/

## 从一个BUG开始

理论上说，任何一款支持事务的数据库都应该支持ACID四个特性。实际上呢，可能因为未知的软件Bug、机器故障（比特反转）等，导致数据库在实际运行中未能完全满足，这就有可能对上层业务带来严重的影响。

比如Percona（MySQL源码的一个分支）修复了一个MySQL可能丢失数据的bug：https://docs.percona.com/percona-server/8.0/release-notes/8.0.39-30.html#bug-fixes。  

关于Bug的更详细解读见：https://www.modb.pro/db/1869263663966728192  

对应的修复代码：https://github.com/percona/percona-server/pull/5385/files

![Img](/images/learn-distributed-system-testing-from-tidb.md/img-20241231103211.png)

上面列举的是一个软件层面BUG导致数据丢失的case，所以从PR中可以看到添加了对应的测试case以确保同一个问题不会再犯。那么其他workload会不会还有类似问题？机器故障时又会有什么表现呢？

## 数据库测试的手段

下面列举了一些数据库测试需要做的一些事情，内容主要来自上面推荐的几篇文章。

### 1、事务正确性测试

首先需要证明数据库实现的事务模型是没有问题的，尤其是在并发读写时，类似的工作已经有人在做了：

**事务隔离级别测试**:

- **Elle**: 用来验证数据库事务隔离级别的检查工具。Elle 是一个纯黑盒的测试工具，巧妙的构造了一个测试场景，通过客户端生成的历史构造出依赖关系图，通过判断依赖图中是否有环以及分析环来确定事务的出现的异常类型，来确定事务的隔离级别。[5] https://github.com/jepsen-io/elle

**事务一致性测试**:

- **Porcupine**： 一个用 Go 实现的线性一致性验证工具。是基于 P-compositionality 算法，P-compositionality 算法利用了线性一致性的 Locality 原理，即如果一个调用历史的所有子历史都满足线性一致性，那么这个历史本身也满足线性一致性。因此，可以将一些不相关的历史划分开来，形成多个规模更小的子历史，转而验证这些子历史的线性一致性。在 TiPocket  有许多 Case 中使用了 Pocupine 检查器 去检查生成的历史，从而判断 TiDB 是否满足线性一致性的约束。 [5] https://github.com/anishathalye/porcupine

### 2、构造丰富的测试case

怎么在你睡觉的时候发现 Bug ？

“我们现在很有意思的一个事情是，迄今为止 PingCAP 没有一个测试人员，这是在所有的公司看来可能都是觉得不可思议的事情，那为什么我们要这么干？因为我们现在的测试已经不可能由人去测了。究竟复杂到什么程度呢？我说几个基本数字大家感受一下：我们现在有六百多万个 Test，这是完全自动化去跑的。然后我们还有大量从社区收集到的各种 ORM Test，一会我会提到这一点。就是这么多 Test 已经不可能是由人写出来的了，以前的概念里面是 Test 是由人写的，但实际上 Test 不一定是人写的，Test 也是可以由机器生成的。举个例子，如果给你一个合法的语法树，你按照这个语法树去做一个输出，比如说你可以更换变量名，可以更换它的表达式等等，你可以生成很多的这种 SQL 出来。[2]”

“但是这地方又蹦出另外一个问题，就是你生成了合法的 SQL 语句，但是你不知道它语句执行的结构，那你怎么去判断它是不是对的？当然业界有很聪明的人。我把它扔给几个数据库同时跑一下，然后取几个大家一致的结果，那我就认为这个结果基本上是对的。如果一个语句过来，然后在我这边执行的结果和另外几个都不一样，那说明我这边肯定错了。就算你是对的，可能也是错的，因为别人执行下来都是这个结果，你不一样，那大家都会认为你是错的。”[2]

### 3、错误注入

一个软件运行在操作系统之上，使用了硬件上的磁盘、CPU、内存、网卡、时钟等资源，在软件上还有文件系统、网络协议栈等。如何证明或者验证在这些依赖的资源出现错误时，数据库系统不会出现不符合ACID的问题？

如果你说把磁盘弄坏、网线拔掉、主机烧毁来模拟这类故障，这的确是个办法，但不是明智的的办法。

![Img](/images/learn-distributed-system-testing-from-tidb.md/img-20241231134035.png)

如果直接损坏硬件成本比较大，不妨加个中间层，这次添加的是故障模拟。“磁盘是模拟的，网络是模拟的，那我们可以监控它，你可以在任何时间、任何的场景下去注入各种错误，你可以注入任何你想要的错误。[3]”

在这方面有很多项目已经做过了，如：

- libfiu：Fault injection in userspace https://blitiri.com.ar/p/libfiu/
- Netflix：《 Failure Injection Testing 》 https://netflixtechblog.com/fit-failure-injection-testing-35d8e2a9bb2
- OpenStack fault-injection library: https://pypi.org/project/os-faults/
- Jepsen: A framework for distributed systems verification, with fault injection  https://github.com/jepsen-io/jepsen
- FoundationDB：Simulation and Testing https://apple.github.io/foundationdb/testing.html
- CharybdeFS: ScyllaDB fault injection filesystem https://github.com/scylladb/charybdefs
- Namazu：Programmable fuzzy scheduler for testing distributed systems https://github.com/osrg/namazu
- american fuzzy lop: a security-oriented fuzzer https://github.com/google/AFL
- OSS-Fuzz：Continuous Fuzzing for Open Source Software https://github.com/google/oss-fuzz

### 4、做好观测

错误可以注入到数据库中了，那么怎么观测数据库服务再哪个环节出现了问题呢？这就得请出 Metrics、Log 和 Tracing 三剑客了。 这些想必大家都很熟悉了：

- Metrics： 丰富的内核指标、秒级数据采集、智能的异常判断&报警

![Img](/images/learn-distributed-system-testing-from-tidb.md/img-20241231135113.png)

- Log： 存放了详细的错误信息，可以使用 FluentBit 或 Promtail，将这些数据导入 ES 或 LOKI 进行相关分析。对于分布式系统，一个事务会请求到做个组件，那么可以使用transaction ID打印到日志中将其关联起来。[4]

![Img](/images/learn-distributed-system-testing-from-tidb.md/img-20241231135106.png)

- Tracing： 在调用链概念出现之前，日志是用来帮助我们理解应用程序中发生情况的唯一途径。大多数的应用程序在他们运行的服务器上创建日志。然而，对于分布式系统来说，光靠日志是不够的，因为光靠日志定位问题的具体位置是一项巨大的挑战。但是调用链则可以非常方便的处理这种场景，因为其可以完整的追踪一个请求的开始到结束所经过的所有节点。[1]

![Img](/images/learn-distributed-system-testing-from-tidb.md/img-20241231135059.png)

### 5、小结

- TiDB作为一个后起之秀，在SQL测试上借助了社区现有 ORM Test，以此保证兼容性
- 同时，利用语法树随机生成SQL并在多款数据库上运行，验证在 TiDB 的SQL执行结果跟其他数据库一致
- 对于分布式系统，利用错误注入方式模拟故障，验证数据库稳定性、可靠性
- 另外还有本文未提及的其他大量测试手段 https://github.com/PingCAP-QE

## PingCAP/TiDB 的测试

上面提及的“事务正确性测试”、“构造丰富的测试case”，PingCAP 开发了一个 tipocket 框架，对其做了集成；在“错误注入”方面，PingCAP 开发了一个对应的 chaos-mesh 混沌工程平台；“做好观测”方面，tidb在设计之初就做了考量，在内核中做了Metrics、Log、Tracing工作，并且可以存储到k8s提供的相关服务中。最后tidb利用argo-workflow来做自动化运行。

- **chaos-mesh**: 混沌工程平台，提供丰富的故障模拟类型，具有强大的故障场景编排能力，方便用户在开发测试中以及生产环境中模拟现实世界中可能出现的各类异常[6] https://github.com/chaos-mesh/chaos-mesh
- **TiDB operator**： Kubernetes 上的 TiDB 集群自动运维系统，提供包括部署、升级、扩缩容、备份恢复、配置变更的 TiDB 全生命周期管理。借助 TiDB Operator，TiDB 可以无缝运行在公有云或自托管的 Kubernetes 集群上。[7] https://github.com/pingcap/tidb-operator
- **tipocket**：一个专门用于测试 TiDB 的测试工具包，它封装了一些同样适合测试其他数据库的测试工具。设计灵感来源于分布式系统领域著名的库jepsen-io/jepsen。https://github.com/pingcap/tipocket
- **argo**：一个基于k8s的工作流平台。比如在运行测试case前，创建namespace、新建secert，然后运行测试case，在测试完成后删除ns等。https://github.com/argoproj/argo-workflows

最终的运行关系如图所示：

![Img](/images/learn-distributed-system-testing-from-tidb.md/img-20250104234735.png)

- 测试人员提交一个测试任务（argo workflow），依次执行step 1、2、3。
- 在执行 step 3 时，运行的镜像是由 tipocket 框架编写而来的测试case。这个case会创建一个tidb集群（节点副本数可以在配置），然后运行测试workload（模拟银行转账、模拟财务报表分析、或者随机生成SQL等，需要根据不同的workload编写对应的数据校验逻辑），workload运行期间还会随机生成各种错误并注入到集群中。
- 直到 workload 运行结束，执行 step 4、5，结束测试。

比如一个简单银行转账测试case包括：

- 初始化N个账户，每个账户M元
- 启动X个线程，模拟并发转账。随机选择出发起人、收款人，转账金额。开启事务修改两个账户余额，并记录转账流水。
- 定时sum所有账户的总额，预期值为 N*M 元，如果总金额不对，立刻退出
- 否则等到测试时间结束

此外还可以设计更复杂的case，比如

- 这期间定时执行 alter table 语句，对于分布式数据库可能还有其他特殊DDL，都可以加入
- 支持转账给多人
- 也可以将转账数据同时写入MySQL等其他数据库，验证不同数据库的结果
- 还可以将转账记录按照对应格式记录到磁盘上，然后利用Elle、Porcupine验证数据前后关系是否符合一致性

## 参考

- [1] https://developers.weixin.qq.com/community/develop/article/doc/00040a73f2440837740fab41b5b413
- [2] https://cn.pingcap.com/blog/distributed-system-test-1/
- [3] https://cn.pingcap.com/blog/distributed-system-test-2/
- [4] https://cn.pingcap.com/blog/distributed-system-test-3/
- [5] https://cn.pingcap.com/blog/building-a-distributed-test-platform-based-on-chaos-mesh-and-argo/
- [6] https://chaos-mesh.org/docs/
- [7] https://docs.pingcap.com/zh/tidb-in-kubernetes/dev/tidb-operator-overview
- [8] https://chaos-mesh.org/zh/blog/building_automated_testing_framework/

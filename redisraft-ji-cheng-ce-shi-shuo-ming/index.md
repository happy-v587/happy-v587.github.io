# RedisRaft - 集成测试说明


## 关于 redisraft 集成测试说明
---

### 两个核心的类

- 通过 `class RedisRaft` 将每个节点的 `init`、`start`、`restart`、`pause`、`join`、`transfer_leader`、`cleanup` 等行为进行管理（通过对 pylib subprocess 、redis cmd 的封装实现）
- 通过 `class Cluster` 将多个 `RedisRaft` 进行管理，包括集群的 `create`、`add_node`、`remove_node`、`pause_leader` 等行为（通过对redis cmd、RedisRaft类的封装实现）

有上面两个给力的 `class` 后，在执行测试case时对集群的生命周期管理就变得非常简单了。

### 测试case的实现

对于每个测试case而言，只需要初始化一个 `cluster`，然后根据需要 `add_node`、`remove_node`；根据需要做 `set key`、`get key` 等操作；根据需要做 `raft COMPACT`等各种操作

- 比如想要测试 raft snapshot 功能是否可以，其中一个case实现是：先启动一个 node 写些数据，然后 raft compact，之后在启动一个新 node，从这个新node查数据，验证是否存在
![image](https://github.com/OpenAtomFoundation/pikiwidb-raft/assets/20750625/c724b2c9-7a9b-4050-82a9-f07f5b83c8f8)

- 比如想要测试 raft membership 功能是否稳定，其中一个case实现是：先创建一个3节点的集群，删掉1号节点，通过2号节点查询节点数是否变成2
![image](https://github.com/OpenAtomFoundation/pikiwidb-raft/assets/20750625/9da23ddf-d3d0-4a61-962a-c6113c572b2a)

从测试case看，redisraft 主要从上层去关注集群状态是否符合预期。而且这些操作都是通过 raft cmd 去完成



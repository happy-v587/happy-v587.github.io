# RedisRaft - 单元测试说明


代码目录

<img width="399" alt="image" src="https://github.com/user-attachments/assets/5ca030a5-0489-42de-b18e-cf3bdd915919">

主要测试了：
- util（对parse_slots函数测试）
- file（对原生read、write、flush、get、sync的包裹）
- log（用来维护 raft log的实现）
- serialization（对 redis command 序列化与反序列化的测试）

<img width="453" alt="image" src="https://github.com/user-attachments/assets/96f836f4-0807-4d75-b3ea-4181c7a513b5">



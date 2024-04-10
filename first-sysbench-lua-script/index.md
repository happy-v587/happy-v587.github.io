# 编写你的第一个 Sysbench Lua Script


## 零、sysbench 介绍

Sysbench 是一个开源的<span style="color: rgb(255, 76, 65);">多线程性能测试工具</span>。它提供了一系列的测试用例，用户<span style="color: rgb(255, 76, 65);">可以通过命令行参数自定义测试的各种方面，如线程数、测试时长、数据大小</span>等。

Sysbench <span style="color: rgb(255, 76, 65);">内置了一些基本的测试模式</span>，如CPU、内存和I/O性能测试，以及针对MySQL和PostgreSQL的数据库测试。但是，当这些内置测试不足以满足用户的特定需求时，Lua 脚本就显得尤为重要。<span style="color: rgb(255, 76, 65);">通过编写Lua脚本，用户可以创建完全定制的测试案例</span>，从而获得更精确和相关的性能指标。

## 一、准备工作：

用简短的时间了解些lua语法：https://www.runoob.com/lua/lua-tutorial.html


## 二、开始编写

编写 sysbench 的 lua 脚本主要工作就是实现如下4个函数，其中init可以不实现

```lua
function init()
    print("init ...")
end

-- sysbench preapre 阶段
function prepare()
    print("prepare ...")
end

-- sysbench cleanup 阶段
function cleanup()
  print("clenup ...")
end

-- sysbench run 阶段，分成3部分，主要逻辑在event中实现，init 和 done 可以用来做数据准备和计算、数据库连接与断开
function thread_init()
  print("thread init ...")
end
function event()
  print("event ...")
end
function thread_done()
  print("thread done ...")
end
```

执行 sysbench prepare 
```bash
#( 04/10/24@ 2:55下午 )( root@MacBook-Pro ):~/code/cpp/sysbench@heads/1.0.20✗✗✗
   sysbench demo.lua  --mysql-host=127.0.0.1 --time=1 --report-interval=1 --mysql-port=4000 --mysql-userroot --mysql-passwordroot --mysql-db=b --table-size=100 --tables=1 --threads=1 prepare
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

prepare ...       # 这里对应 function prepare()
```

执行 sysbench run 
```bash
#( 04/10/24@ 2:55下午 )( root@MacBook-Pro ):~/code/cpp/sysbench@heads/1.0.20✗✗✗
   sysbench demo.lua  --mysql-host=127.0.0.1 --time=1 --report-interval=1 --mysql-port=4000 --mysql-userroot --mysql-passwordroot --mysql-db=b --table-size=100 --tables=1 --threads=1 run 
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

init ...          # 这里对应 function init()
Running the test with following options:
Number of threads: 1
Report intermediate results every 1 second(s)
Initializing random number generator from current time


Initializing worker threads...

thread init ...   # 这里对应 function thread_init()
Threads started!

event ...         # 这里对应 function event()
event ...
... ...
event ...
event ...
thread done ...   # 这里对应 function thread_done()

General statistics:
    total time:                          1.0006s
    total number of events:              680596

Latency (ms):
         min:                                    0.00
         avg:                                    0.00
         max:                                    0.98
         95th percentile:                        0.00
         sum:                                  835.41

Threads fairness:
    events (avg/stddev):           680596.0000/0.00
    execution time (avg/stddev):   0.8354/0.00
```

执行 sysbench cleanup 
```bash
#( 04/10/24@ 2:55下午 )( root@MacBook-Pro ):~/code/cpp/sysbench@heads/1.0.20✗✗✗
   sysbench demo.lua  --mysql-host=127.0.0.1 --time=1 --report-interval=1 --mysql-port=4000 --mysql-userroot --mysql-passwordroot --mysql-db=b --table-size=100 --tables=1 --threads=1 cleanup
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

clenup ...       # 这里对应 function cleanup()
```


当然如果你有一些自定义参数，也可以通过如下方式声明，string、number、boolean都可以，这样就可以在上面的五个函数中使用

```lua
sysbench.cmdline.options = {
   table_size = {"Number of rows per table", 10000},
   create_secondary = {"Create a secondary index in addition to the PRIMARY KEY", true},
   mysql_storage_engine = {"Storage engine, if MySQL is used", "innodb"},
   -- ...
}

-- ...

function event()
  print(sysbench.opt.table_size)   -- 使用自定义命令行参数
end

```

当然 sysbenc 也提供了一些库函数、类型，我们可以直接使用

```lua
-- 一些随机函数
function sysbench.rand.uniform_uint64()
function sysbench.rand.default(a, b)
function sysbench.rand.uniform(a, b)
function sysbench.rand.gaussian(a, b)
function sysbench.rand.special(a, b)
function sysbench.rand.pareto(a, b)
function sysbench.rand.unique()
function sysbench.rand.string(fmt)
function sysbench.rand.uniform_double()

-- 数据库驱动
function sysbench.sql.driver(driver_name)

-- 连接数据库（上面函数执行后返回值可以调动的方法）
local driver_methods = {}
function driver_methods.connect(self)

-- 数据库操作（上面函数执行后返回值可以调动的方法）
local connection_methods = {}
function connection_methods.disconnect(self)
function connection_methods.reconnect(self)
function connection_methods.check_error(self, rs, query)
function connection_methods.query(self, query)
function connection_methods.bulk_insert_init(self, query)
function connection_methods.bulk_insert_next(self, val)
function connection_methods.bulk_insert_done(self)
function connection_methods.prepare(self, query)
function connection_methods.query_row(self, query)

-- 数据库查询结果（执行query后返回值可以调用的方法）
local result_methods = {}
function result_methods.fetch_row(self)
function result_methods.free(self)

-- ...
```

## 三、看个例子

下面看个简单的例子，来自官方 bulk_insert.lua
```lua
#!/usr/bin/env sysbench

-- sysbench prepare 阶段，灌数据
function prepare()
   local i
   local drv = sysbench.sql.driver()
   local con = drv:connect()                                                     -- 连接数据库
   for i = 1, sysbench.opt.threads do
      print("Creating table 'sbtest" .. i .. "'...")                             -- 创建表
      con:query(string.format([[                                                 
        CREATE TABLE IF NOT EXISTS sbtest%d (
          id INTEGER NOT NULL,
          k INTEGER DEFAULT '0' NOT NULL,
          PRIMARY KEY (id))]], i))                                               -- 查询数据，对应 function connection_methods.query(self, query)
   end
end

-- sysbench run 阶段，准备工作
function thread_init()
   drv = sysbench.sql.driver()                                                   -- 连接数据库
   con = drv:connect()
end

-- sysbench run 阶段，收尾工作
cursize=0
function event()
   if (cursize == 0) then
      con:bulk_insert_init("INSERT INTO sbtest" .. thread_id+1 .. " VALUES")     -- function connection_methods.bulk_insert_init(self, query)
   end
   cursize = cursize + 1
   con:bulk_insert_next("(" .. cursize .. "," .. cursize .. ")")                 -- 写入数据，function connection_methods.bulk_insert_next(self, val)
end

-- sysbench run 阶段，收尾工作
function thread_done(thread_9d)
   con:bulk_insert_done()                                                        -- function connection_methods.bulk_insert_done(self)
   con:disconnect()                                                              -- 断开数据库连接
end

-- sysbench cleanup 阶段，清理数据
function cleanup()
   local i
   local drv = sysbench.sql.driver()
   local con = drv:connect()     
   for i = 1, sysbench.opt.threads do
      print("Dropping table 'sbtest" .. i .. "'...")
      con:query("DROP TABLE IF EXISTS sbtest" .. i )
   end
end
```

## 四、跑起来

```lua
-- prepare
sysbench /usr/local/share/sysbench/bulk_insert.lua --mysql-host=127.0.0.1 --time=60 --report-interval=1 --mysql-port=3306 --mysql-user=root --mysql-password=123456 --mysql-db=a --table-size=100 --tables=4 --threads=4 prepare

-- run
sysbench /usr/local/share/sysbench/bulk_insert.lua --mysql-host=127.0.0.1 --time=60 --report-interval=1 --mysql-port=3306 --mysql-user=root --mysql-password=123456 --mysql-db=a --table-size=100 --tables=4 --threads=4 run

-- cleanup
sysbench /usr/local/share/sysbench/bulk_insert.lua --mysql-host=127.0.0.1 --time=60 --report-interval=1 --mysql-port=3306 --mysql-user=root --mysql-password=123456 --mysql-db=a --table-size=100 --tables=4 --threads=4 cleanup
```


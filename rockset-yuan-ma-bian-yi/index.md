# Rockset： 源码编译


> 本文由原作者归档自知乎，原文发布于 2022-03-12。原文链接：[Rockset： 源码编译](https://zhuanlan.zhihu.com/p/480023126)。

## 说明

官方README.md里面已经有详细的编译说明了。值得注意的是：编译之前要先安装aws的sdk，并且要求安装到/usr/local目录下。为了支持存储到s3上，需要在环境变量里面配置USE_AWS=1。

![图 1](/images/rockset-yuan-ma-bian-yi.md/img-01.jpg)

## 编译

### aws-sdk-cpp

> 此处建议使用rockset fork出来的仓库，因为aws官方仓库会不断更新，有可能导致rockset不兼容高版本sdk。

本人编译时就发现clone_example一直运行不起来，排查了一天，最后发现是请求S3时调用PutObject接口因为验签问题报错了。

```bash
git clone https://github.com/rockset/aws-sdk-cpp.git
cd aws-sdk-cpp
git submodule update --init --recursive
mkdir build
cd build
cmake ..
make -j8
make install
```

### rockset-cloud

```bash
git clone https://github.com/rockset/rocksdb-cloud.git
cd rocksdb-cloud
export USE_AWS=1
make static_lib -j 8
cd cloud/examples
make all
```

## 运行

![图 2](/images/rockset-yuan-ma-bian-yi.md/img-02.jpg)


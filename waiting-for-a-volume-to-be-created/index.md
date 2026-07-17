# 使用docker desktop软件开启k8s后创建pvc报错


> 本文由原作者归档自 CSDN，原文发布于 2021-03-10。原文链接：[waiting for a volume to be created, either by external provisioner “docker.io/hostpath“ or manually](https://blog.csdn.net/LIUHUAN0520/article/details/114643684)。

## 背景

使用docker desktop软件开启k8s后，创建pvc报错，详情如下：

waiting for a volume to be created, either by external provisioner " docker .io/hostpath" or manually created by system administrator

## 解决

经过排查发现是文件夹权限问题

[https://github.com/docker/for-mac/issues/3488](https://github.com/docker/for-mac/issues/3488)

添加指定目录然后回车，即可

![图 1](/images/waiting-for-a-volume-to-be-created.md/img-01.png)


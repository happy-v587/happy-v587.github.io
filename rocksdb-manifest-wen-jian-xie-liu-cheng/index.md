# rocksdb：manifest 文件写流程


> 本文由原作者归档自知乎，原文发布于 2024-09-24。原文链接：[rocksdb：manifest 文件写流程](https://zhuanlan.zhihu.com/p/721898389)。

## 零、说明

manifest文件是rocksdb用来管理元信息变动的文件。文件中以VersionEdit为单位。记录了所有SST文件的创建、删除、层级调整等变动行为。

我们都知道，写文件之前需要先将文件open，然后才能写入。所以下文会先介绍manifest文件时怎么open的，然后说明一次变动是如何写入manifest文件中的。

## 一、open 流程

### 1.1、一句话概述

一个fd被三次封装，最后以log::Writer提供使用。

```cpp
descriptor_fname -> WritableFile(在这里open得到fd) -> WritableFileWriter -> log::Writer
```

### 1.2、引用关系

```cpp
std::string descriptor_fname  // 文件名

-> std::unique_ptr<WritableFile> descriptor_file = new PosixWritableFile(descriptor_fname)
// WritableFile 顺序写入的文件抽象。如果实现这个类：需要支持”缓冲，小片段追加到文件，上层控制flush、sync操作”。
// PosixWritableFile 是其中一种实现。还有 PosixMmapFile 等实现

-> std::unique_ptr<WritableFileWriter> file_writer(new WritableFileWriter(std::move(descriptor_file));
//  WritableFile 的包装，添加了 io stat 、 写事件回调 、 限速 、 buffer write 或者 direct write 等功能

-> log::Writer descriptor_log_(new log::Writer(std::move(file_writer), 0, false));
//  具体的业务，log字节流如何组织
```

### 1.3、代码逻辑

- 先准备 EnvOptions
- 然后基于descriptor_fname得到WritableFile
- 再基于WritableFile得到WritableFileWriter
- 最后基于WritableFileWriter得到log::Writer

```cpp
    // 优化 env 参数：写 manifest 时需要 use_mmap_writes=false; use_direct_writes = false; fallocate_with_keep_size = true;
    EnvOptions opt_env_opts = env_->OptimizeForManifestWrite(env_options_);

    // 生成文件名 descriptor_fname
    std::string descriptor_fname = DescriptorFileName(dbname_, pending_manifest_file_number_);

    // 实例化 WritableFile
    std::unique_ptr<WritableFile> descriptor_file;
    s = NewWritableFile(env_, descriptor_fname, &descriptor_file, opt_env_opts);
    descriptor_file->SetPreallocationBlockSize(db_options_->manifest_preallocation_size);

    // 将 WritableFile 转为 WritableFileWriter
    std::unique_ptr<WritableFileWriter> file_writer(new WritableFileWriter(std::move(descriptor_file), descriptor_fname, opt_env_opts, env_,nullptr, db_options_->listeners));

    // 将 WritableFileWriter 转为 log::Writer
    descriptor_log_.reset(new log::Writer(std::move(file_writer), 0, false));
```

## 二、write 流程

### 2.1 一句话概述

先写内存、再进行flush、sync。

### 2.2 代码逻辑

- manifest记录变动的最小单位是 version_edit，所以需要将 edit->EncodeTo(&record)
- 然后将 record 写入 descriptor_log_，即调用 log::Writer的AddRecord
- 最后每次变动都要执行 SyncManifest 确保数据落盘

```cpp
// 将 version edit 写入 manifest
Status VersionSet::ProcessManifestWrites(...) {
    // ...
    for (auto& e : batch_edits) {
      e->EncodeTo(&record);
      s = descriptor_log_->AddRecord(record);   // 逐次 append
    }
    s = SyncManifest(env_, db_options_, descriptor_log_->file());  // 都写完后一把sync
    // ...
}

// 先写
Status Writer::AddRecord(const Slice& slice) {
    // ...
    EmitPhysicalRecord(type, ptr, fragment_length);
    dest_->Flush();
    // ...
}

// 再 sync
Status SyncManifest(...) {
  return file->Sync(db_options->use_fsync);
}

// 写 record 时，分为两次：先写 header、再写 payload。 secondary 读到的数据是空，是读取到了 header 字段判断发现 type 和 length 是 0。
Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr, size_t n) {
  // ...
  Status s = dest_->Append(Slice(buf, header_size));
  if (s.ok()) {
    s = dest_->Append(Slice(ptr, n));
  }
  // ...
}

// 整个写流程都是buffer io，这个 buffer 是 rocksdb 自己维护的一个。当 buffer 不够后将 buffer 数据写到 file，然后 flush file。
Status WritableFileWriter::Append(const Slice& data) {
  // ...
  writable_file_->PrepareWrite(static_cast<size_t>(GetFileSize()), left); // 写 manifest 时此特性已经关闭
  // ...
  buf_.Append(src, left);
  // ...
  Flush();
  // ...
}

Status WritableFileWriter::Flush() {
  // ...
  s = WriteBuffered(buf_.BufferStart(), buf_.CurrentSize());
  // ...
  s = writable_file_->Flush();   // Status PosixWritableFile::Flush() { return Status::OK(); }
}

Status WritableFileWriter::WriteBuffered(const char* data, size_t size) {
  //...
  s = writable_file_->Append(Slice(src, allowed));
  // ...
}

Status PosixWritableFile::Append(const Slice& data) {
  // ...
  const char* src = data.data();
  size_t nbytes = data.size();
  if (!PosixWrite(fd_, src, nbytes)) {
    return IOError("While appending to file", filename_, errno);
  }
  filesize_ += nbytes;
  // ...
}

bool PosixWrite(int fd, const char* buf, size_t nbyte) {
  const size_t kLimit1Gb = 1UL << 30;
  const char* src = buf;
  size_t left = nbyte;
  while (left != 0) {
    size_t bytes_to_write = std::min(left, kLimit1Gb);
    ssize_t done = write(fd, src, bytes_to_write);          // 写数据到fs
    if (done < 0) {
      if (errno == EINTR) {
        continue;
      }
      return false;
    }
    left -= done;
    src += done;
  }
  return true;
}

Status WritableFileWriter::Sync(bool use_fsync) {
  // ...
  Flush();
  // ...
  SyncInternal(use_fsync);
  // ...
}

Status WritableFileWriter::SyncInternal(bool use_fsync) {
  // ...
  writable_file_->Sync();
  // ...
}

Status PosixWritableFile::Sync() {
  // ...
  fdatasync(fd_);                                          // sync 数据
  // ...
}
```

## 总结

1. rocksdb在文件io操作这块做了层层封装和抽象，简化了上层代码的使用方式
2. 这样的抽象也使得rocksdb可以很容易适配不同操作系统(win、linux、macos)
3. 还可以支持更多的存储类型：如硬盘、hdfs、s3等
4. 还可以实现更多的特性：如文件镜像(同时写多份）、模拟文件故障(flush sync做随机故障)


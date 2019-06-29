---
layout:     post
rewards: false
title:      概念
categories:
    - rocksdb
---

# what is rocksdb

基于google leveldb 持久化kv存储，**Keys and values are arbitrary byte
arrays**. 任意大小字节流。(value size must be smaller than 4GB) Log Structured Database Engine for storage， 

- 专为希望在本地或远程存储系统上存储多达数TB数据的应用程序服务器而设计。
- 优化用于在快速存储 - 闪存设备或内存中存储中小尺寸键值
- 它适用于具有多个内核的处理器

# 基本结构

logfile，memtable, sstfile

memtable 是一个内存数据结构，新写入的数据被插入到 memtable 中，并**可选地写入日志文件**。日志文件是存储上顺序写入的文件。

当 memtable 填满时，它被 flush 到存储上的 sstfile ，然后可以被安全地删除。sstfile 中的数据顺序存放，以方便按 key 进行查找。


# datebase

数据库名对应于文件系统的目录的名称，数据库的所有内容都存储该目录下

**RocksDB Options**可以随时更改 `can be changed dynamically while DB is
running`, **automatically keeps** options used in the database in **OPTIONS-xxxx files** under the DB directory.

## create rocksdb 坑

```python
import rocksdb


_CONFIG = CONFIG['databases']['rocksdb']
_DB_LOG = {
    'db_log_dir': _CONFIG['log_dir'],
    'max_log_file_size': _CONFIG['max_log_size'],
    'keep_log_file_num': _CONFIG['max_log_num'],
}


def create_rocksdb_client(db_name: str, read_only: bool = True) -> rocksdb.DB:
    # 不能放在外面 [fix] Exception: Options object is already used by another DB
    _READONLY_OPTIONS = rocksdb.Options(
        create_if_missing=True,
        **_DB_LOG,
    )

    _WRITABLE_OPTIONS = rocksdb.Options(
        create_if_missing=True,
        max_background_compactions=4,
        max_background_flushes=4,
        allow_mmap_writes=True,
        compression=rocksdb.CompressionType.lz4_compression,
        **_DB_LOG,
    )

    if read_only:
        return rocksdb.DB(os.path.join(_CONFIG['data_dir'], db_name), _READONLY_OPTIONS, read_only=read_only)
    return rocksdb.DB(os.path.join(_CONFIG['data_dir'], db_name), _WRITABLE_OPTIONS, read_only=read_only)
```
- 只有 `read_only=True` rocksdb 可以多开 不然就会报`IO Error Lock: No locks available` 
- `Options object is already used by another DB`
  一个程序里面一个option貌似只能给一个db
- 设置了`read_only=True`，`create_if_missing=True`就没什么鬼用。

# 原子操作

```python
batch = rocksdb.WriteBatch()
batch.put(b"key", b"v1")
batch.delete(b"key")
batch.put(b"key", b"v2")
batch.put(b"key", b"v3")

db.write(batch)
```

`WriteBatch` 有序执行

# 写入
**默认**情况下，每次写入rocksdb都是**异步**的：它在将进程写入操作系统后返回。从操作系统内存到底层持久存储的转移是异步发生的。
sync可以为特定写入打开该标志，以使写入操作不会返回，直到正在写入的数据一直被推送到持久存储。

对于非同步写入 RocksDB only buffers WAL write in OS buffer or internal
buffer 它们通常比同步写入快得多。非同步写入的缺点是机器崩溃可能导致最后几次更新丢失。
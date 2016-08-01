# dsdfs - A dead simple distributed file system w/ a focus on speed.

## Introduction

There are a lot of distributed file systems aimed at high avilability. See:
  - Glusterfs
  - Ceph
  - HDFS
  - ... and many more

All those systems are very powerful, but sacrafice write speed for the sake of reliabiilty:
  - Glusterfs shards a file across many nodes in the cluster (by default, other configurations exists, see below).
  - Ceph, same thing (N>=1)
  - HDFS writes locally but creates 2 remote copies (N=3)

In these systems, a write is considered complete IFF it is sharded and distributed.
Let's make a file system does away with high avilability / reliability for the sake of write speed - consider a write complete once it is written locally. All other nodes will see the file, and read it directly off the remote disk if needed.

"But what if that node fails?" You ask. "Then the file is gone" - we say. That's it. No funny buisness. Tradeoffs exists, and here is one.

"Why would this every be useful? I like not losing files!" Well, because sometimes we don't care about reliably storing files. Some file storage needs can be part of ephemeral workloads, and are only important to keep if the task completes:
  - Computing ML models
  - Compiling a large binary for testing
  - Intermidate data for Hadoop jobs
  - Staging INSERT data

So you see, "one size does not fit all". Let's make a distributed file system that is dead simple, focuses on write speed, and always moves bytes lazily. Here's a rough outline of the usage:
  - `N nodes` in a system, all mounting dsdfs at `/shared` - initially empty
  - `Node 1` writes a file `/shared/somefile`, and is now "owner" of `somefile`.
  - `Node 2` can run `ls /shared` and sees `somefile`.
  - `Node 2` can run `cat /shared/somefile`, which will read the file directly from Node 1.
    - `Node 2` will then cache this file with its timestamp locally.
  - `Node 2` can execute `echo hello >> /shared/somefile`, which will write some bytes to the file on `Node 1`.
    - This is because `Node 1` is considered the "owner" of the file. These bytes cannot be shareded across the two nodes.
    - In contrast, `echo hello > /shared/somefile` would be a local write, as it would overwrite the file on `Node 1`.
How will it work? See the next section.

## Architecture

DSDFS is based on FUSE, and backed by actual local disk for local writes.

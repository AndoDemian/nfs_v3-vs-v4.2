# NFS v3 vs v4.2 - Nothing to write home about
### _Date published: 2020-04-26_

---

## Setup

**NFS-Server**
```
OS: Ubuntu 18.04.4
Kernel: 4.15.0-1065-aws
NFS-Kernel-Server: 1:1.3.4-2.1ubuntu5.2
AWS Instance Type: c5.xlarge
Disk: ext4 / GP2 - 335 GB / 1005 IOPS / 250 MBs/sec Throughput
MTU: 9001
```

</br>

**NFS-Client**
```
OS: Ubuntu 18.04.4
Kernel: 4.15.0-1065-aws
NFS-Kernel-Server: 1:1.3.4-2.1ubuntu5.2
AWS Instance Type: c5.xlarge
MTU: 9001
```

---

## Scope

**NFS-Server**

Options: `rw,async,all_squash,no_subtree_check,anonuid=33,anongid=33`  
- rw: The client machine will have read and write access to the directory.
- async: File data changes are made initially to memory. This speeds up performance but is more likely to result in data loss.
- allsquash: Squash every remote user, including root.
- nosubtreecheck: If only part of a volume is exported, a routine called subtree checking verifies that a file that is requested from the client is in the appropriate part of the volume. If the entire volume is exported, disabling this check will speed up transfers.
- anonuid/anongid: These options explicitly set the uid and gid of the anonymous account. Set anonymous user/group id to a specific id.

</br>

**NFS-Client**
```bash
v3: rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=tcp,timeo=14,retrans=3,sec=sys,mountvers=3,mountport=36327,mountproto=udp,local_lock=none

v3_perf: rw,noatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,nolock,noacl,proto=tcp,timeo=14,retrans=3,sec=sys,mountvers=3,mountport=57309,mountproto=udp,local_lock=all

v4.2: rw,relatime,vers=4.2,rsize=32768,wsize=32768,namlen=255,hard,proto=tcp,timeo=14,retrans=3,sec=sys,local_lock=none

v4.2_perf: rw,noatime,vers=4.2,rsize=32768,wsize=32768,namlen=255,hard,proto=tcp,timeo=14,retrans=3,sec=sys,local_lock=none
```    

Options:
- rw:  Mount the filesystem read-write.
- relatime: Update inode access times relative to modify or change time. Access time is only updated if the previous access time was earlier than the current modify or change time.
- noatime: Do not update inode access times on this filesystem. This works for all inode types (directories too), so it implies nodiratime.
- rsize: The maximum number of bytes in each network READ request that the NFS client can receive when reading data from a file on an NFS server.
- wsize: The maximum number of bytes per network WRITE request that the NFS client can send when writing data to a file on an NFS server.
- namlen: The maximum length of a pathname component on this mount. If this option is not specified, the maximum length is negotiated with the server. In most cases, this maximum length is 255 characters.
- hard:  The hard option will cause the mount to continue to retry until the server responds.
- nolock: Applications can lock files, but such locks provide exclusion only against other applications running on the same client.
- noacl: Selects whether to use the NFSACL sideband protocol on this mount point.
- proto: The transport protocol name and protocol family the NFS client uses to transmit requests to the NFS server for this mount point.
- timeo: The time in deciseconds (tenths of a second) the NFS client waits for a response before it retries an NFS request.
- retrans: The number of times the NFS client retries a request before it attempts further recovery action.
- sec: The RPCGSS security flavor to use for accessing files on this mount point. If the sec option is not specified, or if sec=sys is specified, the NFS client uses the AUTH_SYS security flavor for all NFS requests on this mount point.
- mountproto: The transport protocol name and protocol family the NFS client uses to transmit requests to the NFS server's mountd service when performing this mount request, and when later unmounting this mount point.
- local_lock: Specifies whether to use local locking for any or both of the flock and the POSIX locking mechanisms. mechanism can be one of all, flock, posix, or none. This option is supported in kernels 2.6.37 and later. If all is specified, the client assumes that both flock and POSIX locks are local. If this option is not specified, or if none is specified, the client assumes that the locks are not local.

---

## Findings

Before starting to throw numbers in your face, lemme start by saying that there are a lot of NFS benchmarking tools, but there isn't a "go-to" tool far as I've seen.

### IOzone

Command: 
```bash
iozone -l 2 -u 2 -r 64k -s 8G -F /path/file1 /path/file2
```

```
Record Size 64 kB
File size set to 8388608 kB
Time Resolution = 0.000001 seconds.
Processor cache size set to 1024 kBytes.
Processor cache line size set to 32 bytes.
File stride size set to 17 * record size.
Min process = 2
Max process = 2
Throughput test with 2 processes
Each process writes a 8388608 kByte file in 64 kByte records
```

#### Results IOZone (kBytes/sec):
```
    +----------------+-----------+-------------+-----------+---------------+
    |                | NFS_v3    | NFS_v3_perf | NFS_v4.2  | NFS_v4.2_perf |
    +----------------+-----------+-------------+-----------+---------------+
    | Initial write  | 278122.41 |   276829.28 | 277119.72 |     276224.92 |
    +----------------+-----------+-------------+-----------+---------------+
    | Rewrite        |  273184.5 |   273765.23 |  273887.2 |     273208.94 |
    +----------------+-----------+-------------+-----------+---------------+
    | Read           | 255104.07 |   252774.13 | 254674.86 |     254843.91 |
    +----------------+-----------+-------------+-----------+---------------+
    | Re-read        |    260368 |   259820.66 | 260383.47 |     260377.79 |
    +----------------+-----------+-------------+-----------+---------------+
    | Reverse Read   | 183664.37 |   150949.12 | 188689.35 |     156792.35 |
    +----------------+-----------+-------------+-----------+---------------+
    | Stride read    | 157970.11 |   153495.46 |  156372.3 |     151984.55 |
    +----------------+-----------+-------------+-----------+---------------+
    | Random read    | 150265.03 |   122811.32 | 152790.74 |     123342.19 |
    +----------------+-----------+-------------+-----------+---------------+
    | Mixed workload | 303571.64 |   300393.83 |  301718.4 |      300947.5 |
    +----------------+-----------+-------------+-----------+---------------+
    | Random write   | 274635.42 |   273754.73 | 278107.72 |     276879.05 |
    +----------------+-----------+-------------+-----------+---------------+
    | Pwrite         | 280013.14 |    278870.5 | 277356.08 |      276379.8 |
    +----------------+-----------+-------------+-----------+---------------+
    | Pread          |  255346.6 |   255210.17 | 254182.77 |     254209.84 |
    +----------------+-----------+-------------+-----------+---------------+
    | Fwrite         | 278649.45 |   276834.86 | 277127.62 |     276365.97 |
    +----------------+-----------+-------------+-----------+---------------+
    | Fread          | 254851.98 |    254903.7 | 254830.31 |     255375.05 |
    +----------------+-----------+-------------+-----------+---------------+
``` 

**IOZone Jobs Description** (*from original documentation*)

**Initial write:** This test measures the performance of writing a new file. When a new file is written, not only does the data need to be stored but also the overhead information for keeping track of where the data is located on the storage media. This overhead is called metadata. It consists of the directory information, the space allocation, and any other data associated with a file that is not part of the data contained in the file.

**Rewrite:** This test measures the performance of writing a file that already exists. When a file is written that already exists, the work required is less as the metadata already exists.

**Read:** This test measures the performance of reading an existing file.

**Re-read:** This test measures the performance of reading a file that was recently read. It is normal for the performance to be higher as the operating system generally maintains a cache of the data for files that were recently read.

**Reverse Read:** This test measures the performance of reading a file backwards.

**Stride read:** This test measures the performance of reading a file with a strided access behavior. An example would be: Read at offset zero for a length of 4 Kbytes, then seek 200 Kbytes, and then read for a length of 4 Kbytes, then seek 200 Kbytes and so on.

**Random read:** This test measures the performance of reading a file with accesses being made to random locations within the file.

**Mixed workload:** This test measures the performance of reading and writing a file with accesses being made to random locations within the file. The distribution of read/write is done on a round-robin basis. More than one thread/process is required for proper operation.

**Random write:** This test measures the performance of writing a file with accesses being made to random locations within the file.

**Pwrite:** This test measures the performance of writing data on a particular position in a file.

**Pread:** This test measures the performance of reading data on a particular position in a file.

**Fwrite:** This test measures the performance of writing a file using the library function fwrite(). This is a library routine that performs buffered write operations.

**Fread:** This test measures the performance of reading a file using the library function fread(). This is a library routine that performs buffered & blocked read operations.

---

### FIO
```bash
    # Sequential Reads – Async mode – 8K block size – Direct IO – 100% Reads
    fio --filename=/path/seqreadfile --name=seqread --rw=read --direct=1 --ioengine=libaio --bs=8k --numjobs=8 --size=8G --runtime=300  --group_reporting
    # Sequential Writes – Async mode – 32K block size – Direct IO – 100% Writes
    fio --filename=/path/seqwritefile --name=seqwrite --rw=write --direct=1 --ioengine=libaio --bs=32k --numjobs=8 --size=8G --runtime=300 --group_reporting
    # Random Reads – Async mode – 8K block size – Direct IO – 100% Reads
    fio --filename=/path/randreadfile --name=randread --rw=randread --direct=1 --ioengine=libaio --bs=8k --numjobs=8 --size=8G --runtime=300 --group_reporting
    # Random Writes – Async mode – 64K block size – Direct IO – 100% Writes
    fio --filename=/path/randwritefile --name=randwrite --rw=randwrite --direct=1 --ioengine=libaio --bs=64k --numjobs=8 --size=8G --runtime=300 --group_reporting
    # Random Read/Writes – Async mode – 16K block size – Direct IO – 75% Reads/25% Writes
    fio --filename=/path/randrwfile --name=randrw --rw=randrw --direct=1 --ioengine=libaio --bs=16k --numjobs=8 --rwmixread=75 --size=8G --runtime=300 --group_reporting
```    

#### Results FIO
```
    +-------------------------------------------------------------------------------------+
    |          Random Reads – Async mode – 8K block size – Direct IO – 100% Reads         |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |                     |      NFS_3 |   NFS_3perf |   NFS_4.2 |  NFS_4.2perf |
    +---------+---------------------+------------+-------------+-----------+--------------+
    | read    |   iops (total used) |    4132    |     4135    |    4087   |     4100     |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |     bandwidth (avg) |  33.9MB/s  |   33.9MB/s  |  33.5MB/s |   33.6MB/s   |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |       latency (avg) |   1934.9   |   1933.25   |  1955.89  |    1949.96   |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          iops (avg) |   516.76   |    517.89   |   511.19  |    512.82    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          io (total) |   10.2GB   |    10.2GB   |   10.0GB  |    10.1GB    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |                                                                                     |
    +-------------------------------------------------------------------------------------+
    | Random Read/Writes – Async mode – 16K block size – Direct IO – 75% Reads/25% Writes |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |                     |      NFS_3 |   NFS_3perf |   NFS_4.2 |  NFS_4.2perf |
    +---------+---------------------+------------+-------------+-----------+--------------+
    | read    |   iops (total used) |       3519 |        3524 |      3539 |         3557 |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |     bandwidth (avg) |  57.7MB/s  |   57.7MB/s  |  57.0MB/s |   58.3MB/s   |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |       latency (avg) |   2180.01  |    2193.4   |  2168.31  |    2174.95   |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          iops (avg) |   440.04   |    440.57   |   442.47  |     445.3    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          io (total) |   17.3GB   |    17.3GB   |   17.4GB  |    17.5GB    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    | write   |   iops (total used) |    1174    |     1175    |    1180   |     1186     |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |     bandwidth (avg) |  19.2MB/s  |   19.3MB/s  |  19.3MB/s |   19.4MB/s   |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |       latency (avg) |   272.61   |    224.34   |   270.18  |    216.42    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          iops (avg) |    146.8   |    146.98   |   147.59  |    148.52    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          io (total) |   5773MB   |    5779MB   |   5804MB  |    5834MB    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |                                                                                     |
    +-------------------------------------------------------------------------------------+
    |        Random Writes – Async mode – 64K block size – Direct IO – 100% Writes        |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |                     | NFS_3      | NFS_3perf   | NFS_4.2   | NFS_4.2perf  |
    +---------+---------------------+------------+-------------+-----------+--------------+
    | write   |   iops (total used) |    4464    |     4481    |    4483   |     4497     |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |     bandwidth (avg) |   293MB/s  |   294MB/s   |  294MB/s  |    295MB/s   |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |       latency (avg) |   1788.33  |    1781.8   |  1779.87  |    1774.96   |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          iops (avg) |   558.57   |    560.33   |   561.13  |    562.83    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          io (total) |   68.7GB   |    68.7GB   |   68.7GB  |    68.7GB    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |                                                                                     |
    +-------------------------------------------------------------------------------------+
    |        Sequential Reads – Async mode – 8K block size – Direct IO – 100% Reads       |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |                     | NFS_3      | NFS_3perf   | NFS_4.2   | NFS_4.2perf  |
    +---------+---------------------+------------+-------------+-----------+--------------+
    | read    |   iops (total used) |    30.8k   |    34.4k    |   30.8k   |     35.9k    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |     bandwidth (avg) |   252MB/s  |   282MB/s   |  252MB/s  |    294MB/s   |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |       latency (avg) |   258.35   |    230.31   |   258.21  |    220.89    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          iops (avg) |   3854.61  |   4316.82   |  3854.41  |    4500.02   |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          io (total) |   68.7GB   |    68.7GB   |   68.7GB  |    68.7GB    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |                                                                                     |
    +-------------------------------------------------------------------------------------+
    |      Sequential Writes – Async mode – 32K block size – Direct IO – 100% Writes      |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |                     |    NFS_3   |  NFS_3perf  |  NFS_4.2  |  NFS_4.2perf |
    +---------+---------------------+------------+-------------+-----------+--------------+
    | write   |   iops (total used) |    18.5k   |    18.6k    |   18.4k   |     18.8k    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |     bandwidth (avg) |   606MB/s  |   611MB/s   |  602MB/s  |    615MB/s   |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |       latency (avg) |   431.26   |    426.97   |    434    |    425.01    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          iops (avg) |   2312.34  |    2331.9   |  2297.66  |    2346.4    |
    +---------+---------------------+------------+-------------+-----------+--------------+
    |         |          io (total) |   68.7GB   |    68.7GB   |   68.7GB  |    68.7GB    |
    +---------+---------------------+------------+-------------+-----------+--------------+
```    

--- 

## Conclusion

> The tests conducted are tailored to show multiple general situations.

From my point of view, the outcome is clear. There is no absolute winner, and it's highly dependent on how you are using NFS in your environment.  
Will the parameters detailed above will work for you? The short answer is "No".  

Identify your needs ( Writing / Reading, Small / Big Files, Sequential / Random ), test, tune parameters, test, and then ... test some more.
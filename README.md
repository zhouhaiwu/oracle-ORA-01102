# oracle-ORA-01102
错误 ORA-01102: cannot mount database in EXCLUSIVE mode 的处理方法
-----------------------
今天启动数据库时报错了！

SQL> startup mount        

ORACLE instance started.

 

Total System Global Area  608174080 bytes

Fixed Size                 1220844 bytes

Variable Size            176164628 bytes

Database Buffers      427819008 bytes

Redo Buffers             2969600 bytes

ORA-01102: cannot mount database in EXCLUSIVE mode

 

Google了一下发现一个写的非常好的帖子，详细内如如下（被我修改过了！）

 

分析原因：

一、在HA系统中，已经有其他节点启动了实例，将双机共享的资源（如磁盘阵列上的裸设备）占用了；

 

二、说明Oracle被异常关闭时，有资源没有被释放，一般有以下几种可能，

1、 Oracle的共享内存段或信号量没有被释放；

2、 Oracle的后台进程（如SMON、PMON、DBWn等）没有被关闭；

3、 用于锁内存的文件lk<sid>和sgadef<sid>.dbf文件没有被删除。

 

解决思路：

当发生1102错误时，可以按照以下流程检查、排错：

如果是HA系统，检查其他节点是否已经启动实例检查Oracle进程是否存在，如果存在则杀掉进程检查信号量是否存在，如果存在，则清除信号量检查共享内存段是否存在，如果存在，则清除共享内存段检查锁内存文件lk<sid>和sgadef<sid>.dbf是否存在，如果存在，则删除。
 

具体做法：

首先，虽然我们的系统是HA系统，但是备节点的实例始终处在关闭状态，这点通过在备节点上查数据库状态可以证实。

其次、是因系统掉电引起数据库宕机的，系统在接电后被重启，因此我们排除了第二种可能种的1、2点。最可疑的就是第3点了。

查$ORACLE_HOME/dbs目录：

$ cd $ORACLE_HOME/dbs

$ ls sgadef*

sgadef* not found

$ ls lk*

/opt/oracle/product/ 10.2.0/db_1/dbs/lkSIMPLY

lkSIMPLY

果然，lk<sid>文件没有被删除。将它删除掉

$ rm lk*

 

再次启动时又遇到下面的错误，不过别担心，继续后面的操作就搞定

SQL> startup mount

ORACLE instance started.

 

Total System Global Area  608174080 bytes

Fixed Size    1220844 bytes

Variable Size     176164628 bytes

Database Buffers      427819008 bytes

Redo Buffers    2969600 bytes

ORA-00205: error in identifying control file, check alert log for more info   : (

 

查看共享内存段

[root@simply bdump]# ipcs -map

 

------ Shared Memory Creator/Last-op --------

shmid   owner  cpid    lpid

786444  root    6490   6438

819213  root    6549   6438

1409040 oracle   31502  16728

根据ID号清楚共享内存段

ipcrm –m 1409040

我这里操作是没有成功的，不过执行了下面的操作就ok了！

 

查看信号量

[root@simply bdump]# ipcs -s

 

key       semid      owner   perms    nsems

0x17ff6454 360448     oracle    640     154

 

清除oracle的信号量

[root@simply bdump]# ipcrm -s 360448

 

再次查询确认

[root@simply bdump]# ipcs -s

 

------ Semaphore Arrays --------

key  semid  owner  perms   nsems

 

再查询共享内存段也ok了！

[root@simply bdump]# ipcs -m

 

如果是Oracle进程没有关闭，

$kill -9 <PID>

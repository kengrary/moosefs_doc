# MooseFS
##MooseFS概述

###优势：
  - Free(GPL)
  - 通用文件系统，不需要修改上层应用就可以使用
  - 可以在线扩容，体系架构可伸缩性极强。
  - 部署简单。
  - 体系架构高可用，所有组件无单点故障。
  - 文件对象高可用，可设置任意的文件冗余程度。
  - 提供Windows回收站的功能。
  - 提供类似Java语言的 GC（垃圾回收）。
  - 提供snapshot特性。
  - google filesystem的一个c实现。
  - 提供web gui监控接口。

###可能的瓶颈：
  - master本身的性能瓶颈。
  - 体系架构存储文件总数的可遇见的上限。（mfs把文件系统的结构缓存到master的内存中，个人认为文件越多，master的内存消耗越大，8g对应2500kw的文件数，2亿文件就得64GB内存 ）。
  - 单点故障解决方案的健壮性。


###读写过程


 
 
 
##MooseFS安装
- 从github下载最新的MooseFS源代码包
```
$ wget https://codeload.github.com/moosefs/moosefs/zip/master
```
- MooseFS编译依赖`fuse`，需要安装`fuse`和`fuse-devel`
```
$ yum install –y fuse fuse-devel
```
- 解压MooseFS源代码，执行
```
$ ./configure –prefix=/app/mfs 
$ make
$ make install
```
- 编译完成并安装到/app/mfs目录，目录中主要有以下内容：
 - sbin：MooseFS服务的可执行文件
 - bin：MooseFS常用命令执行文件
 - var：MooseFS服务运行时产生的文件
 - etc：MooseFS服务配置文件

##MooseFS配置
- 配置master server，master负责各个数据存储服务器的管理,文件读写调度，文件空间回收以及恢复、多节点拷贝。
```
cp /app/mfs/etc/mfs/mfsmaster.cfg.sample /app/mfs/etc/mfs/mfsmaster.cfg
cp /app/mfs/etc/mfs/mfsexport.cfg.sample /app/mfs/etc/mfs/mfsexport.cfg.sample
```
- 修改`mfsmaster.cfg`文件，主要配置以下信息
```
WORKING_USER = nobody
WORKING_GROUP = nobody
SYSLOG_IDENT = mfsmaster
LOCK_MEMORY = 0
DATA_PATH = /root/mfs/var/mfs
EXPORTS_FILENAME = /root/mfs/etc/mfs/mfsexports.cfg
BACK_LOGS = 50
```
 
- 修改`mfsexport.cfg`文件，可以根据实际需要限制可以访问IP地址
```
# Allow everything but "meta".
11.111.0.0/16                       /       rw,alldirs,admin,maproot=0:0
# Allow "meta".
11.111.0.0/16                       .       rw
```
- 启动master server，mfsmaster启动后监听9419,9420,9421端口
```
sbin/mfsmaster
open files limit has been set to: 16384
working directory: /root/mfs/var/mfs
lockfile created and locked
initializing mfsmaster modules ...
exports file has been loaded
mfstopology configuration file (/root/mfs/etc/mfstopology.cfg) not found - using defaults
loading metadata ...
loading sessions data ... ok (0.0000)
loading storage classes data ... ok (0.0000)
loading objects (files,directories,etc.) ... ok (0.0889)
loading names ... ok (0.0659)
loading deletion timestamps ... ok (0.0000)
loading quota definitions ... ok (0.0000)
loading xattr data ... ok (0.0000)
loading posix_acl data ... ok (0.0000)
loading open files data ... ok (0.0000)
loading flock_locks data ... ok (0.0000)
loading posix_locks data ... ok (0.0000)
loading chunkservers data ... ok (0.0000)
loading chunks data ... ok (0.0600)
checking filesystem consistency ... ok
connecting files and chunks ... ok
all inodes: 4
directory inodes: 1
file inodes: 3
chunks: 32
metadata file has been loaded
stats file has been loaded
master <-> metaloggers module: listen on *:9419
master <-> chunkservers module: listen on *:9420
main master server module: listen on *:9421
mfsmaster daemon initialized properly
```

- 配置chunk server，chunk server负责连接管理服务器，听从管理服务器调度，提供存储空间，并为客户提供数据传输。
```
cp /app/mfs/etc/mfs/ mfschunkserver.cfg.sample /app/mfs/etc/mfs/ mfschunkserver.cfg
cp /app/mfs/etc/mfs/ mfshdd.cfg.sample /app/mfs/etc/mfs/ mfshdd.cfg
```
修改`mfschunkserver.cfg`，主要配置master server的IP和端口
```
MASTER_HOST = 11.111.0.9
MASTER_PORT = 9420
```
修改`mfshdd.cfg`，主要配置用于存储数据的路径和可使用的空间
/root/data 10G
- 启动chunk server，chunk server启动后监听9422端口
```
sbin/mfschunkserver
open files limit has been set to: 16384
working directory: /root/mfs/var/mfs
lockfile created and locked
setting glibc malloc arena max to 4
setting glibc malloc arena test to 4
initializing mfschunkserver modules ...
hdd space manager: path to scan: /root/data/
hdd space manager: start background hdd scanning (searching for available chunks)
main server module: listen on *:9422
stats file has been loaded
mfschunkserver daemon initialized properly
```

- client端挂载MooseFS的文件系统
```
bin/mfsmount /mnt/mfs -H  <master server ip>
```
mount成功后，mount命令显示以下信息
```
11.111.0.9:9421 on /mfsdata type fuse.mfs (rw,nosuid,nodev,allow_other)
```
- 启动web管理界面，web管理界面启动后监听9425端口
```
sbin/mfscgiserv
lockfile created and locked
starting simple cgi server (host: any , port: 9425 , rootpath: /root/mfs/share/mfscgi)
```
- 配置metalogger server（备份master server）
```
cp /app/mfs/etc/mfs/mfsmetalogger.cfg.sample /app/mfs/etc/mfs/mfsmetalogger.cfg
```
修改`mfsmetalogger.cfg`文件，主要修改master server的IP和端口
```
MASTER_HOST = 11.111.0.9
MASTER_PORT = 9419
```
- 启动metalogger server
```
sbin/mfsmetalogger
open files limit has been set to: 4096
working directory: /root/mfs/var/mfs
lockfile created and locked
initializing mfsmetalogger modules ...
mfsmetalogger daemon initialized properly
``` 
##MooseFS特性配置
- 设置文件被拷贝的份数，拷贝份数一般小于或等于chunk server数量，如果设置超过chunk server数量，则只会产生等于chunk server数量的拷贝，-r参数是对目录进行递归设置拷贝数
```
bin/mfssetgoal –r 2 /path
```
- 显示设置的拷贝数
```
bin/mfsgetgoal /path
```
- 设置回收站保留被删除文件的时间，默认值为86400秒，即1天
```
bin/mfssettrashtime 300 /path
```
显示回收站保留被删除文件的时间
```
bin/mfsgettrashtime /path
```
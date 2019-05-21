### 1. 设置和查看lustre参数
- 创建文件系统时，使用mkfs.lustre。
- 当服务器停止运行时，使用use trnefs.lustre。
- 当文件系统正在运行时，使用lctl设置或者查看参数。


#### 1.1 mkfs.lustre设置参数
- 当文件系统第一次格式化时，参数可以通过在mkfs.lustre命令中添加--param选项进行设置。

```bash
# 设置超时时间为50
mkfs.lustre --mdt --param="sys.timeout=50" /dev/sda
```

#### 1.2 tunefs.lustre设置参数
- 当服务器(OSS或MDS)停止运行时，可通过tunefs.lustre命令及--param选项添加参数至现有文件系统。


```bash
#tunefs.lustre命令添加的为新的参数，而不会替代参数。
tunefs.lustre --param==failover.node=192.168.0.13@tcp0 /dev/sda

#擦除所有的已有参数并添加新的参数
tunefs.lustre --erase-params --param=new_parameters

#用户可以设置任何在/proc/fs/lustre文件中可设置的具有OBD设备的参数，可指定为*obdname|fsname*. *obdtype*.*proc_file_name*=*value*
tunefs.lustre --param mdt.identity_upcall=NONE /dev/sda1
```

#### 1.3 lctl设置参数
- 当文件系统运行时，lctl可用于设置参数(临时或永久)。
    - a. 临时参数在服务器或者客户端未关闭时处于激活状态。
    - b. 永久参数在服务器和客户端重启后仍不变。
    
##### 1.3.1 设置临时参数

```bash
#列出所有可设置参数
lctl list_param

#lctl set_param设置当前运行节点上的临时参数。这些参数映射至/proc/{fs,sys}/{lnet,lustre}
lctl set_param osc.*.max_dirty_mb=1024
```
 
##### 1.3.2 设置永久参数

```bash
#使用lctl conf_param设置永久参数。可用于设置/proc/fs/lustre文件中所有可设置的参数。(参数持久化到MGS文件系统配置中)
lctl conf_param testfs-MDT0000.sys.timeout=40

#使用lctl set_param -P设置永久参数。(必须在MGS上执行)
lctl set_param -P osc.*.max_dirty_mb=1024

#使用lctl set_param删除永久参数。(用-d删除永久参数)
lctl set_param -P -d osc.*.max_dirty_mb

```
 
#### 1.4 列出可设置的参数

```bash
#列出可设置的参数

lctl list_param ost.OSS.ost.*
ost.OSS.ost.high_priority_ratio
ost.OSS.ost.nrs_crrn_quantum
```

#### 1.5 查看参数值

```bash
#查看当前参数值
lctl get_param ost.OSS.ost.threads_max
ost.OSS.ost.threads_max=30
```


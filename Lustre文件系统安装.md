# 1. 环境
## 1.1 创建临时的yum源
```
cat >/tmp/lustre-repo.conf <<\__EOF
[lustre-server]
name=lustre-server
baseurl=https://downloads.whamcloud.com/public/lustre/latest-release/el7/server
# exclude=*debuginfo*
gpgcheck=0

[lustre-client]
name=lustre-client
baseurl=https://downloads.whamcloud.com/public/lustre/latest-release/el7/client
# exclude=*debuginfo*
gpgcheck=0

[e2fsprogs-wc]
name=e2fsprogs-wc
baseurl=https://downloads.whamcloud.com/public/e2fsprogs/latest/el7
# exclude=*debuginfo*
gpgcheck=0
__EOF
```
## 1.2 下载相关软件包
```
mkdir -p /var/www/html/repo
cd /var/www/html/repo
reposync -c /tmp/lustre-repo.conf -n \
-r lustre-server \
-r lustre-client \
-r e2fsprogs-wc
```
## 1.3 创建repo
```
cd /var/www/html/repo
for i in e2fsprogs-wc lustre-client lustre-server; do
(cd $i && createrepo .)
done
```
## 1.4 lustre repo
```
hn=`hostname --fqdn`
cat >/var/www/html/lustre.repo <<__EOF
[lustre-server]
name=lustre-server
baseurl=https://$hn/repo/lustre-server
enabled=0
gpgcheck=0
proxy=_none_

[lustre-client]
name=lustre-client
baseurl=https://$hn/repo/lustre-client
enabled=0
gpgcheck=0

[e2fsprogs-wc]
name=e2fsprogs-wc
baseurl=https://$hn/repo/e2fsprogs-wc
enabled=0
gpgcheck=0
__EOF
```
## 1.5 查看lustre repo
```
yum repolist all
```

# 2. 安装
## 2.1 安装Lustre Server软件
### 2.1.1. 安装e2fsprogs
```
yum --nogpgcheck --disablerepo=* --enablerepo=e2fsprogs-wc \
install e2fsprogs
```
### 2.1.2. 安装并升级内核
```
yum --nogpgcheck --disablerepo=base,extras,updates \
--enablerepo=lustre-server install \
kernel \
kernel-devel \
kernel-headers \
kernel-tools \
kernel-tools-libs \
kernel-tools-libs-devel
```

### 2.1.3. 重启
```
reboot
```

### 2.1.4. 安装ldiskfs kmod和lustre包
```
yum --nogpgcheck --enablerepo=lustre-server install \
kmod-lustre \
kmod-lustre-osd-ldiskfs \
lustre-osd-ldiskfs-mount \
lustre \
lustre-resource-agents
```
### 2.1.5  加载lustre到内核
```
modprobe -v lustre
modprobe -v ldiskfs
```

## 2.2 安装lustre client
### 2.2.1 升级内核
```
yum install \
kernel \
kernel-devel \
kernel-headers \
kernel-abi-whitelists \
kernel-tools \
kernel-tools-libs \
kernel-tools-libs-devel
```
### 2.2.2 重启
```
reboot
```
### 2.2.3 安装kmod包
```
yum --nogpgcheck --enablerepo=lustre-client install \
kmod-lustre-client \
lustre-client
```
### 2.2.4  加载lustre到内核
```
modprobe -v lustre
```


# 3.创建lustre文件系统
**配置说明**
*   --fsname：指定生成后的lustre文件系统名，如sgfs，将来客户端采用mount -t 192.168.100.1@tcp0:192.168.100.2@[tcp0:/sgfs](http://tcp0/sgfs) /home进行挂载。
*   --mgs：指定为MGS分区
*   --mgt：指定为MGT分区
*   --ost：指定为OST分区
*   --servicenode=ServiceNodeIP@tcp0：指定本节点失效时，接手提供服务的节点，如为InfiniBand网络，那么tcp0需要换成o2ib
*   --index：指定索引，不能相同


## 3.1 安装MGS
```
#格式化
mkfs.lustre --fsname=lustrefs --reformat --mgs --servicenode=mds1@tcp0  /dev/vdb

#挂载
mount -t lustre /dev/vdb /mnt/mgs
```

## 3.2 安装MDT
```
#格式化
mkfs.lustre --mdt --fsname=lustrefs --index=0 --mgsnode=mds1@tcp0 --servicenode=mds1@tcp0 --reformat /dev/vdb

#开启quota
tune2fs -O project /dev/vdb

#挂载目录
mount -t lustre /dev/vdc /mnt/mdt
```

## 3.3 安装OST
```
#格式化
mkfs.lustre --fsname=lustrefs --ost --reformat --index=0 --servicenode=ost1@tcp0 --servicenode=ost2@tcp0 --mgsnode=mds1@tcp0 /dev/vdb

#开启quota
tune2fs -O project,quota /dev/vdb

mount -t lustre /dev/vdb /mnt/ost1
```

### 启用quota
```
lctl conf_param lustrefs.quota.ost=ugp; 
lctl conf_param lustrefs.quota.mdt=ugp;
```

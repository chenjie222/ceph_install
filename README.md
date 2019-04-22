## CentOS 7部署 Ceph分布式存储架构

#### 1.主机准备

| 主机    | ip            | 环境       |
| ------- | ------------- | ---------- |
| master1 | 10.122.60.157 | admin, mon |
| slave1  | 10.122.60.158 | osd,mds    |
| slave2  | 10.122.60.159 | osd,mds    |
| slave3  | 10.122.60.160 | osd,mds    |

### 编辑host文件

| 10.122.60.157:master1 |
| --------------------- |
| 10.122.60.158:slave1  |
| 10.122.60.159:slave2  |
| 10.122.60.160:slave3  |

### SSH免密码登录

[root@master1~]# ssh-keygen #所有的输入选项都直接回车生成。

[root@master1~]# ssh-copy-id slave1

[root@master1~]# ssh-copy-id slave2

[root@master1~]# ssh-copy-id slave3

[root@master1~]# ssh-copy-id master1

#### ntp安装

```
yum install ntp ntpdate ntp-doc
#查看 /etc/ntp.conf server 源是否一样 可以修改为其他ntp服务器
```

启动ntp

```
systemctl start ntpd
```

查看时间源

```
ntpq -p
```

#### 管理节点安装ceph-deploy工具

- 第一步：增加 yum配置文件（各个节点都需要增加yum源） 

  vim /etc/yum.repos.d/ceph.repo 

  ```
  添加以下内容：(ceph国内163源)
  [Ceph]
  name=Ceph packages for $basearch
  baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/x86_64
  enabled=1
  gpgcheck=1
  type=rpm-md
  gpgkey=http://mirrors.163.com/ceph/keys/release.asc
  priority=1
   
  [Ceph-noarch]
  name=Ceph noarch packages
  baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/noarch
  enabled=1
  gpgcheck=1
  type=rpm-md
  gpgkey=http://mirrors.163.com/ceph/keys/release.asc
  priority=1
   
  [ceph-source]
  name=Ceph source packages
  baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/SRPMS
  enabled=1
  gpgcheck=1
  type=rpm-md
  gpgkey=http://mirrors.163.com/ceph/keys/release.asc
  priority=1
  ```

  

- 第二步：更新软件源并安装ceph-deploy 管理工具 

  ```
  yum clean all && yum list
  
  yum update && yum makecache
  yum clean all && yum list && yum update && yum makecache
  yum -y install ceph-deploy 
  ```



#### 创建monitor服务

```
mkdir /etc/ceph && cd /etc/ceph
ceph-deploy new master1   #mon安装在master1节点
#这时会生成3个配置文件
ceph.conf  ceph.log  ceph.mon.keyring
Ceph配置文件、一个monitor密钥环和一个日志文件
```

#### 修改副本数

vim ceph.conf 

```
[global]
fsid = 5d8ade8d-3d54-441d-abd6-82cf7710b2a0
mon_initial_members = master1
mon_host = 10.122.60.157
auth_cluster_required = cephx    #开启了 cephx 认证。
auth_service_required = cephx
auth_client_required = cephx
osd_pool_default_size = 2      新增 #默认osd少于3个不会正常启动
public_network=10.122.60.0/24   新增 
osd_max_object_name_len = 256    新增 解决ext4文件系统osd不启动问题
osd_max_object_namespace_len = 64   新增 解决ext4文件系统osd不启动问题
osd check max object name len on startup = false   新增 解决ext4文件系统osd不启动问题

```

 cephx 认证 ：从用户空间挂载Ceph 文件系统前，确保客户端主机有一份Ceph 配置副本、和具备 Ceph元数据服务器能力的密钥环。在客户端主机上，把监视器主机上的Ceph 配置文件拷贝到客户端服务器的 /etc/ceph/目录下。 

#### 在所有节点安装ceph

```
ceph-deploy install master1 slave1 slave2 slave3
```

初始化 monitor 节点并收集所有密钥。 

```
ceph-deploy mon create-initial
```

如果执行过程中报错，则加上 --overwrite-conf 参数

```
ceph-deploy --overwrite-conf mon create-initial
```

执行完毕后，会在当前目录下生成一系列的密钥环，应该是各组件之间访问所需要的认证信息吧。 

```
-rw-------. 1 cephd cephd    113 12月  7 15:13 ceph.bootstrap-mds.keyring
-rw-------. 1 cephd cephd     71 12月  7 15:13 ceph.bootstrap-mgr.keyring
-rw-------. 1 cephd cephd    113 12月  7 15:13 ceph.bootstrap-osd.keyring
-rw-------. 1 cephd cephd    113 12月  7 15:13 ceph.bootstrap-rgw.keyring
-rw-------. 1 cephd cephd    129 12月  7 15:13 ceph.client.admin.keyring
-rw-rw-r--. 1 cephd cephd    222 12月  7 14:47 ceph.conf
-rw-rw-r--. 1 cephd cephd 120207 12月  7 15:13 ceph-deploy-ceph.log
-rw-------. 1 cephd cephd     73 12月  7 14:46 ceph.mon.keyring
```

#### 部署osd服务

ceph monitor 已经成功启动了 接下来需要创建 OSD 了 

```
$ ssh slave1
$ sudo mkdir /var/local/osd0
$ sudo chown -R ceph:ceph /var/local/osd0
$ exit

$ ssh slave2
$ sudo mkdir /var/local/osd1
$ sudo chown -R ceph:ceph /var/local/osd1
$ exit

$ ssh slave3
$ sudo mkdir /var/local/osd1
$ sudo chown -R ceph:ceph /var/local/osd1
$ exit
```

注意：这里执行了 `chown -R ceph:ceph` 操作，将 osd0 和 osd1 目录的权限赋予 ceph:ceph，否则，接下来执行 `ceph-deploy osd activate ...` 时会报权限错误。 

#### 创建激活osd

```
#创建osd 
ceph-deploy osd prepare slave1:/var/local/osd0 slave2:/var/local/osd1 slave3:/var/local/osd2
#激活osd 
ceph-deploy osd activate slave1:/var/local/osd0 slave2:/var/local/osd1 slave3:/var/local/osd2
```

如果报错就加  --overwrite-conf 参数

#### 统一配置

通过 `ceph-deploy admin` 将配置文件和 admin 密钥同步到各个节点，以便在各个 Node 上使用 ceph 命令时，无需指定 monitor 地址和 `ceph.client.admin.keyring` 密钥。 

```
ceph-deploy admin slave1 slave2 slave3
```

各节点修改ceph.client.admin.keyring权限 

```
chmod +r /etc/ceph/ceph.client.admin.keyring 
```

#### 查看osd状态

```
ceph health 或 ceph -s
```

### 部署mds服务

```
ceph-deploy mds create slave1 slave2 slave3
#查看mds状态
ceph mds stat
```



```
 [root@slave1 ceph]# ceph -s 
	cluster 5d8ade8d-3d54-441d-abd6-82cf7710b2a0
     health HEALTH_OK
     monmap e1: 1 mons at {master1=10.122.60.157:6789/0}
            election epoch 3, quorum 0 master1
      fsmap e5: 1/1/1 up {0=slave2=up:active}, 2 up:standby  #1个mds在使用 另2个是处于热备份的状态
     osdmap e21: 3 osds: 3 up, 3 in
            flags sortbitwise,require_jewel_osds
      pgmap v1217: 320 pgs, 3 pools, 4411 bytes data, 20 objects
            24784 MB used, 245 GB / 281 GB avail
                 320 active+clean

```

#### 创建ceph文件系统

```
[root@master1]# ceph fs ls   #创建之前
No filesystems enabled
创建存储池
[root@master1]# ceph osd pool create cephfs_data <pg_num> 
[root@master1]# ceph osd pool create cephfs_metadata <pg_num>
```

其中：**<pg_num> = 128 ,**
关于创建存储池
确定 pg_num 取值是强制性的，因为不能自动计算。下面是几个常用的值：
　　*少于 5 个 OSD 时可把 pg_num 设置为 128
　　*OSD 数量在 5 到 10 个时，可把 pg_num 设置为 512
　　*OSD 数量在 10 到 50 个时，可把 pg_num 设置为 4096
　　*OSD 数量大于 50 时，你得理解权衡方法、以及如何自己计算 pg_num 取值
　　*自己计算 pg_num 取值时可借助 pgcalc 工具
随着 OSD 数量的增加，正确的 pg_num 取值变得更加重要，因为它显著地影响着集群的行为、以及出错时的数据持久性（即灾难性事件导致数据丢失的概率）。 

#### 创建文件系统

创建好存储池后，你就可以用 fs new 命令创建文件系统了

```
[root@master1]# ceph fs new <fs_name> cephfs_metadata cephfs_data 
其中：<fs_name> = cephfs  可自定义
 
[root@master1]# ceph fs ls              #查看创建后的cephfs
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
[root@master1]# ceph mds stat          #查看mds节点状态
e5: 1/1/1 up {0=slave2=up:active}, 2 up:standby 
active是活跃的，另2个是处于热备份的状态
```

### 挂载Ceph文件系统

- #### 内核驱动挂载Ceph文件系统

```
[root@master1 ceph]# mkdir /cep #创建挂载点
存储密钥（如果没有在管理节点使用ceph-deploy拷贝ceph配置文件）
cat /etc/ceph/ceph.client.admin.keyring
将key对应的值复制下来保存到文件：/etc/ceph/admin.secret中。
挂载
[root@xuegod70 ceph]# mount -t ceph 10.10.10.67:6789:/ /cep -o name=admin,secretfile=/etc/ceph/admin.secret
取消挂载
[root@xuegod70 ceph]# umount /cep
```

- #### 用户控件挂载Ceph文件系统

  ```
  安装ceph-fuse
  [root@xuegod70 ceph]# yum install -y ceph-fuse
  挂载
  [root@xuegod70 ceph]# ceph-fuse -m 10.122.60.157:6789 /cep
  取消挂载
  [root@xuegod70 ceph]# fusermount -u /cep
  #
  ```

以上这样均需安装ceph客户端，如果不用客户端，直接使用nfs即可

#### nfs挂载ceph文件系统

部署nfs

1、安装nfs服务器

```
 yum install nfs-utils -y
```

2、将/cep策略写入/etc/exports

[root@node3 mycephfs]#echo “/mnt/mycephfs   *(rw,async,no_root_squash,no_subtree_check)” >> /etc/exports

3、执行exportfs -a生效

[root@node3 mycephfs]#exportfs -a

4、nfs、rpcbind服务设置自启

[root@node3 mycephfs]#service rpcbind restart

 [root@node3 mycephfs]#service nfs restart

 [root@node3 mycephfs]#chkconfig nfs on

1. 其他node挂载nfs服务器的/cep即可

#### 卸载ceph

```
ps aux|grep ceph |awk '{print $2}'|xargs kill -9
ps -ef|grep ceph
umount /var/lib/ceph/osd/*
rm -rf /var/lib/ceph/osd/*
rm -rf /var/lib/ceph/mon/*
rm -rf /var/lib/ceph/mds/*
rm -rf /var/lib/ceph/bootstrap-mds/*
rm -rf /var/lib/ceph/bootstrap-osd/*
rm -rf /var/lib/ceph/bootstrap-rgw/*
rm -rf /var/lib/ceph/tmp/*
rm -rf /etc/ceph/*
rm -rf /var/run/ceph/*
# 清理配置
ceph-deploy purgedata master1 slave1 slave2 slave3
ceph-deploy forgetkeys

# 清理 Ceph 安装包
ceph-deploy purge master1 slave1 slave2 slave3
```





坑：

1.官网的源千万不能使用， 会导致超时报错，

更换为163源

2.创建osd是报错

```
root@master1:/etc/ceph# ceph-deploy osd prepare master1:/var/local/osd1
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.39): /usr/bin/ceph-deploy osd prepare master1:/var/lib/ceph/osd/ceph-0
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  block_db                      : None
[ceph_deploy.cli][INFO  ]  disk                          : [('master1', '/var/lib/ceph/osd/ceph-0', None)]
[ceph_deploy.cli][INFO  ]  dmcrypt                       : False
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  bluestore                     : None
[ceph_deploy.cli][INFO  ]  block_wal                     : None
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : prepare
[ceph_deploy.cli][INFO  ]  dmcrypt_key_dir               : /etc/ceph/dmcrypt-keys
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f26c5556ef0>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  fs_type                       : xfs
[ceph_deploy.cli][INFO  ]  filestore                     : None
[ceph_deploy.cli][INFO  ]  func                          : <function osd at 0x7f26c59bc500>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  zap_disk                      : False
[ceph_deploy.osd][DEBUG ] Preparing cluster ceph disks master1:/var/lib/ceph/osd/ceph-0:
[master1][DEBUG ] connected to host: master1 
[master1][DEBUG ] detect platform information from remote host
[master1][DEBUG ] detect machine type
[master1][DEBUG ] find the location of an executable
[master1][INFO  ] Running command: /sbin/initctl version
[master1][DEBUG ] find the location of an executable
[ceph_deploy.osd][INFO  ] Distro info: Ubuntu 14.04 trusty
[ceph_deploy.osd][DEBUG ] Deploying osd to master1
[master1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph_deploy.osd][DEBUG ] Preparing host master1 disk /var/lib/ceph/osd/ceph-0 journal None activate False
[master1][DEBUG ] find the location of an executable
[master1][INFO  ] Running command: /usr/sbin/ceph-disk -v prepare --cluster ceph --fs-type xfs -- /var/lib/ceph/osd/ceph-0
[master1][WARNIN] command: Running command: /usr/bin/ceph-osd --cluster=ceph --show-config-value=fsid
[master1][WARNIN] command: Running command: /usr/bin/ceph-osd --check-allows-journal -i 0 --log-file $run_dir/$cluster-osd-check.log --cluster ceph --setuser ceph --setgroup ceph
[master1][WARNIN] command: Running command: /usr/bin/ceph-osd --check-wants-journal -i 0 --log-file $run_dir/$cluster-osd-check.log --cluster ceph --setuser ceph --setgroup ceph
[master1][WARNIN] command: Running command: /usr/bin/ceph-osd --check-needs-journal -i 0 --log-file $run_dir/$cluster-osd-check.log --cluster ceph --setuser ceph --setgroup ceph
[master1][WARNIN] command: Running command: /usr/bin/ceph-osd --cluster=ceph --show-config-value=osd_journal_size
[master1][WARNIN] populate_data_path: Preparing osd data dir /var/lib/ceph/osd/ceph-0
[master1][WARNIN] command: Running command: /bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-0/ceph_fsid.9803.tmp
[master1][WARNIN] command: Running command: /bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-0/fsid.9803.tmp
[master1][WARNIN] command: Running command: /bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-0/magic.9803.tmp
[master1][INFO  ] checking OSD status...
[master1][DEBUG ] find the location of an executable
[master1][INFO  ] Running command: /usr/bin/ceph --cluster=ceph osd stat --format=json
[ceph_deploy.osd][DEBUG ] Host master1 is now ready for osd use.
root@master1:/etc/ceph# 
root@master1:/etc/ceph# ceph-deploy osd activate master1:/var/lib/ceph/osd/ceph-0
[ceph_deploy.conf][DEBUG ] found configuration file at: /root/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (1.5.39): /usr/bin/ceph-deploy osd activate master1:/var/lib/ceph/osd/ceph-0
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : activate
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7fc954721ef0>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function osd at 0x7fc954b87500>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.cli][INFO  ]  disk                          : [('master1', '/var/lib/ceph/osd/ceph-0', None)]
[ceph_deploy.osd][DEBUG ] Activating cluster ceph disks master1:/var/lib/ceph/osd/ceph-0:
[master1][DEBUG ] connected to host: master1 
[master1][DEBUG ] detect platform information from remote host
[master1][DEBUG ] detect machine type
[master1][DEBUG ] find the location of an executable
[master1][INFO  ] Running command: /sbin/initctl version
[master1][DEBUG ] find the location of an executable
[ceph_deploy.osd][INFO  ] Distro info: Ubuntu 14.04 trusty
[ceph_deploy.osd][DEBUG ] activating host master1 disk /var/lib/ceph/osd/ceph-0
[ceph_deploy.osd][DEBUG ] will use init type: upstart
[master1][DEBUG ] find the location of an executable
[master1][INFO  ] Running command: /usr/sbin/ceph-disk -v activate --mark-init upstart --mount /var/lib/ceph/osd/ceph-0
[master1][WARNIN] main_activate: path = /var/lib/ceph/osd/ceph-0
[master1][WARNIN] activate: Cluster uuid is ea99362b-48ca-4e73-90b5-1e48cf7930e7
[master1][WARNIN] command: Running command: /usr/bin/ceph-osd --cluster=ceph --show-config-value=fsid
[master1][WARNIN] activate: Cluster name is ceph
[master1][WARNIN] activate: OSD uuid is 5f99b598-7ab7-47ed-8a3b-61af167b2457
[master1][WARNIN] allocate_osd_id: Allocating OSD id...
[master1][WARNIN] command: Running command: /usr/bin/ceph --cluster ceph --name client.bootstrap-osd --keyring /var/lib/ceph/bootstrap-osd/ceph.keyring osd create --concise 5f99b598-7ab7-47ed-8a3b-61af167b2457
[master1][WARNIN] command: Running command: /bin/chown -R ceph:ceph /var/lib/ceph/osd/ceph-0/whoami.9860.tmp
[master1][WARNIN] activate: OSD id is 0
[master1][WARNIN] activate: Initializing OSD...
[master1][WARNIN] command_check_call: Running command: /usr/bin/ceph --cluster ceph --name client.bootstrap-osd --keyring /var/lib/ceph/bootstrap-osd/ceph.keyring mon getmap -o /var/lib/ceph/osd/ceph-0/activate.monmap
[master1][WARNIN] got monmap epoch 1
[master1][WARNIN] command: Running command: /usr/bin/timeout 300 ceph-osd --cluster ceph --mkfs --mkkey -i 0 --monmap /var/lib/ceph/osd/ceph-0/activate.monmap --osd-data /var/lib/ceph/osd/ceph-0 --osd-journal /var/lib/ceph/osd/ceph-0/journal --osd-uuid 5f99b598-7ab7-47ed-8a3b-61af167b2457 --keyring /var/lib/ceph/osd/ceph-0/keyring --setuser ceph --setgroup ceph
[master1][WARNIN] Traceback (most recent call last):
[master1][WARNIN]   File "/usr/sbin/ceph-disk", line 9, in <module>
[master1][WARNIN]     load_entry_point('ceph-disk==1.0.0', 'console_scripts', 'ceph-disk')()
[master1][WARNIN]   File "/usr/lib/python2.7/dist-packages/ceph_disk/main.py", line 5371, in run
[master1][WARNIN]     main(sys.argv[1:])
[master1][WARNIN]   File "/usr/lib/python2.7/dist-packages/ceph_disk/main.py", line 5322, in main
[master1][WARNIN]     args.func(args)
[master1][WARNIN]   File "/usr/lib/python2.7/dist-packages/ceph_disk/main.py", line 3453, in main_activate
[master1][WARNIN]     init=args.mark_init,
[master1][WARNIN]   File "/usr/lib/python2.7/dist-packages/ceph_disk/main.py", line 3273, in activate_dir
[master1][WARNIN]     (osd_id, cluster) = activate(path, activate_key_template, init)
[master1][WARNIN]   File "/usr/lib/python2.7/dist-packages/ceph_disk/main.py", line 3378, in activate
[master1][WARNIN]     keyring=keyring,
[master1][WARNIN]   File "/usr/lib/python2.7/dist-packages/ceph_disk/main.py", line 2853, in mkfs
[master1][WARNIN]     '--setgroup', get_ceph_group(),
[master1][WARNIN]   File "/usr/lib/python2.7/dist-packages/ceph_disk/main.py", line 2800, in ceph_osd_mkfs
[master1][WARNIN]     raise Error('%s failed : %s' % (str(arguments), error))
[master1][WARNIN] ceph_disk.main.Error: Error: ['ceph-osd', '--cluster', 'ceph', '--mkfs', '--mkkey', '-i', u'0', '--monmap', '/var/lib/ceph/osd/ceph-0/activate.monmap', '--osd-data', '/var/lib/ceph/osd/ceph-0', '--osd-journal', '/var/lib/ceph/osd/ceph-0/journal', '--osd-uuid', u'5f99b598-7ab7-47ed-8a3b-61af167b2457', '--keyring', '/var/lib/ceph/osd/ceph-0/keyring', '--setuser', 'ceph', '--setgroup', 'ceph'] failed : 2018-02-11 13:43:16.470829 7f0a5123a800 -1 filestore(/var/lib/ceph/osd/ceph-0) mkfs: write_version_stamp() failed: (13) Permission denied
[master1][WARNIN] 2018-02-11 13:43:16.470897 7f0a5123a800 -1 OSD::mkfs: ObjectStore::mkfs failed with error -13
<span style="color:#FF0000;">[master1][WARNIN] 2018-02-11 13:43:16.470981 7f0a5123a800 -1  ** ERROR: error creating empty object store in /var/lib/ceph/osd/ceph-0: (13) Permission denied</span>
[master1][WARNIN] 
[master1][ERROR ] RuntimeError: command returned non-zero exit status: 1
[ceph_deploy][ERROR ] RuntimeError: Failed to execute command: /usr/sbin/ceph-disk -v activate --mark-init upstart --mount /var/lib/ceph/osd/ceph-0

```

chown -R ceph:ceph /var/lib/ceph/osd 

3.执行命令：

01 ceph-deploy install admin-node node1 node2 node3

ERROR：[ceph_deploy][ERROR ] RuntimeError: Failed to execute command: rpm -Uvh --replacepkgs 

http://ceph.com/rpm-giant/el6/noarch/ceph-release-1-0.el6.noarch.rpm



02 ceph-deploy install admin-node node1 node2 node3

ERROR: [ceph1][INFO ] Running command: yum -y install ceph ceph-radosgw

[ceph1][WARNIN] No data was received after 300 seconds, disconnecting

[ceph_deploy][ERROR ] RuntimeError: Failed to execute command: ceph --version

 

解决办法：进入/etc/yum.repos.d/ceph.repo文件将所有的源改为网易163源，或者阿里源（经测试网易163源更好用点）编辑ceph源的配置文件， vim  /etc/yum.repo.d/ceph.repo

Ceph官网提供的download.ceph.com在一些情况下非常不好用。在此建议使用网易yum  



4.osd创建也激活了， 始终osd处于inactive状态

ceph osd tree 

```
-1 0.27507 root default                                      
-2 0.09169     host slave1                                   
 0 0.09169         osd.0        down  1.00000          1.00000 
-3 0.09169     host slave2                                   
 1 0.09169         osd.1        down  1.00000          1.00000 
-4 0.09169     host slave3                                   
 2 0.09169         osd.2        down  1.00000          1.00000 
```

解决办法：

```
df-hT
[root@master1 cep]# df -hT
Filesystem           Type      Size  Used Avail Use% Mounted on
devtmpfs             devtmpfs   32G     0   32G   0% /dev
tmpfs                tmpfs      32G   12K   32G   1% /dev/shm
tmpfs                tmpfs      32G  521M   31G   2% /run
tmpfs                tmpfs      32G     0   32G   0% /sys/fs/cgroup
/dev/vda3            ext4       94G  3.9G   87G   5% /
/dev/vda1            ext4      488M  197M  256M  44% /boot
tmpfs                tmpfs     6.3G     0  6.3G   0% /run/user/0
```

linux文件系统为ext4

需要再ceph.conf中增加

```
osd_max_object_name_len = 256
osd_max_object_namespace_len = 64
osd check max object name len on startup = false
```

重新安装osd

```
#创建osd 
ceph-deploy --overwrite-conf osd prepare slave1:/var/local/osd0 slave2:/var/local/osd1 slave3:/var/local/osd2
#激活osd 
ceph-deploy --overwrite-conf osd activate slave1:/var/local/osd0 slave2:/var/local/osd1 slave3:/var/local/osd2
```


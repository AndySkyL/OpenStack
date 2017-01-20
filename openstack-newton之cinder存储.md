##OpenStack-Newton之Cinder存储
OpenStack 支持多种存储配置，如NFS，Ceph, GlusterFS, ISCSI等存储方式，可以根据自己不同业务需求来进行合理的规划。
关于ISCSI的配置，在*OpenStack-newton部署指南*中已经提到过，这里将主要介绍NFS，GlusterFS和Ceph的配置方法。


###Cinder-NFS
继续使用之前的实验环境，我们将node2作为一个存储节点来创建一个NFS存储。

1. 安装需要的软件包，cinder-volume，和nfs:
   
   ```
   yum install openstack-cinder python-keystone -y
   yum install -y nfs-utils rpcbind
   ```

2. 配置nfs挂载：
   
   ```
   mkdir /data/nfs
   echo "/data/nfs  *(rw,no_root_squash)" > /etc/exports
   systemctl  start rpcbind
   systemctl start nfs
   ```

* 配置cinder使用nfs（从之前iscsi的cinder配置直接拷贝到本地，如果没有配置，可以参考openstack官方的cinder配置,并做如下更改）:
  
  ```
  scp /etc/cinder/cinder.conf  172.16.10.32:/etc/cinder/
  ```
   * 编辑`/etc/cinder/cinder.conf`修改`[lvm]`部分的配置项，替换为`[nfs]`。
  
     ```
    [nfs]
	volume_driver = cinder.volume.drivers.nfs.NfsDriver
	nfs_shares_config = /etc/cinder/nfs_shares
	nfs_mount_point_base = $state_path/mnt
   
     ```
   * 修改配置文件`/etc/cinder/cinder.conf`，在`[DEFAULT]`配置后端存储方式为nfs:
     
     ```
     [DEFAULT]
     enabled_backends = nfs
     ```
   
* 创建`/etc/cinder/nfs_shares`文件:
     
     ```
     172.16.10.32:/data/nfs
     ```

* 修改`nfs_shares`权限
     
     ```
     chown  root:cinder /etc/cinder/nfs_shares
     chmod 640 /etc/cinder/nfs_shares
     ```
   
* 启动服务：
  
  ```
  systemctl  enable openstack-cinder-volume
  systemctl start openstack-cinder-volume 
  ```
* 查看本地nfs是否挂载：
  
  ```
  [root@node2 ~]# df -h
  Filesystem               Size  Used Avail Use% Mounted on
  ...
  172.16.10.32:/data/nfs   8.5G  3.9G  4.6G  46% /var/lib/cinder/mnt/c0abd38dcf00ad03918477e4e5cefdee
  
  ```
  
* 在控制节点查看volume是否启动：
  
  ```
  [root@node1 ~]# source admin-openstack 
[root@node1 ~]# openstack volume service list
+------------------+-----------+------+---------+-------+----------------------------+
| Binary           | Host      | Zone | Status  | State | Updated At                 |
+------------------+-----------+------+---------+-------+----------------------------+
| cinder-scheduler | node1     | nova | enabled | up    | 2017-01-19T11:57:02.000000 |
| cinder-volume    | node1@lvm | nova | enabled | up    | 2017-01-19T11:57:01.000000 |
| cinder-volume    | node2@nfs | nova | enabled | up    | 2017-01-19T11:57:01.000000 |
+------------------+-----------+------+---------+-------+----------------------------+
   ```
 
 * 在控制节点配置cinder,创建`NFS`类型:
   
   ```
   [root@node1 ~]# cinder type-create NFS
+--------------------------------------+------+-------------+-----------+
| ID                                   | Name | Description | Is_Public |
+--------------------------------------+------+-------------+-----------+
| 19e20347-fd13-407b-b911-42444a3e9747 | NFS  | -           | True      |
+--------------------------------------+------+-------------+-----------+
  ```
* 创建`ISCSI`类型，（之前已经配置过ISCSI存储）
   
  ```
  [root@node1 ~]# cinder type-create ISCSI
+--------------------------------------+-------+-------------+-----------+
| ID                                   | Name  | Description | Is_Public |
+--------------------------------------+-------+-------------+-----------+
| 51f1547a-39cb-41e7-84dd-0f49a490393e | ISCSI | -           | True      |
+--------------------------------------+-------+-------------+-----------+
  ```

* 将创建的类型和后端存储关联起来，分别修改ISCSI和NFS存储节点上的cinder配置：
  * ISCSI存储节点，添加`volume_backend_name`：
    
    ```
    [lvm]
    volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
    volume_group = cinder-volumes
    iscsi_protocol = iscsi
    iscsi_helper = lioadm
    volume_backend_name = ISCSI-Storage
    ```
    > 提示： 在存储节点加载对应存储的驱动，默认驱动在： `/usr/lib/python2.7/site-packages/cinder/volume/drivers/nfs.py` 目录，需要指定到对应驱动文件的类。
    
  * NFS存储节点，添加`volume_backend_name`：
    
    ```
    [nfs]
    volume_driver = cinder.volume.drivers.nfs.NfsDriver
    nfs_shares_config = /etc/cinder/nfs_shares
    nfs_mount_point_base = $state_path/mnt
    volume_backend_name = NFS-Storage
    ```

* 关联创建的NFS和ISCSI卷类型：
  
  ```
  # cinder type-key NFS set volume_backend_name=NFS-Storage
  # cinder type-key ISCSI set volume_backend_name=ISCSI-Storage 
  ```
* 这样在Horizon中就可以看到有NFS，ISCSI类型的卷类型了。
  
  
####Cinder其它存储类型的创建
对于其它类型的后端存储，与NFS的配置方式流程相差不大，一般会有如下步骤：

* 准备存储节点（如NFS，GlusterFS,LVM等）
* 安装cinder-volume:
  
  ```
  yum install openstack-cinder python-keystone -y
  ```
  
* 配置`/etc/cinder/cinder.conf`,配置驱动并关联对应的存储类型：
  
  ```
  [xxx]
  volume_driver = cinder.volume.drivers.xxx.XXXDriver
  ...
  volume_backend_name = xxx-Storage
  ```
* 启动cinder-volume: 
  
  ```
  systemctl  enable openstack-cinder-volume
  systemctl start openstack-cinder-volume 
  ```
* 创建存储类型：
  
  ```
  cinder type-create xxx
  ```
* 关联类型：
  
  ```
  cinder type-key xxx set volume_backend_name=xxx-Storage
  ```

按照类似的思路，我们可以配置GlusterFS作为后端存储。

####Cinder-GlusterFS

1. GlusterFS的创建和NFS创建相似，在开始之前需要先创建一个GlusterFS的存储集群：
[参考官方文档](https://gluster.readthedocs.io/en/latest/Quick-Start-Guide/Quickstart/)

2. 安装cinder-volume,配置`/etc/cinder/cinder.conf`,配置驱动并关联对应的存储类型：

   ```
   enabled_backends = gluster
   ```

   ```
   [gluster]
   volume_driver = cinder.volume.drivers.glusterfs.GlusterfsDriver
   glusterfs_shares_config = /etc/cinder/glusterfs_shares
   glusterfs_mount_point_base = $state_path/mnt
   volume_backend_name = GlusterFS-Storage
   ```
* 创建`/etc/cinder/glusterfs_shares`文件:
     
     ```
     cat /etc/cinder/glusterfs_shares
     
     Gluster-server1:/gv0
     ```

* 修改`glusterfs_shares`权限
     
     ```
     chown  root:cinder /etc/cinder/glusterfs_shares
     chmod 640 /etc/cinder/glusterfs_shares
     ```
* 启动服务：
  
  ```
  systemctl  enable openstack-cinder-volume
  systemctl start openstack-cinder-volume 
  ```
* 在控制节点查看Gluster卷是否启动：
  
  ```
  [root@node1 ~]# openstack volume service list
+------------------+---------------+------+---------+-------+----------------------------+
| Binary           | Host          | Zone | Status  | State | Updated At                 |
+------------------+---------------+------+---------+-------+----------------------------+
| cinder-scheduler | node1         | nova | enabled | up    | 2017-01-20T11:37:07.000000 |
| cinder-volume    | node1@lvm     | nova | enabled | up    | 2017-01-20T11:37:06.000000 |
| cinder-volume    | node2@gluster | nova | enabled | up    | 2017-01-20T11:37:06.000000 |
+------------------+---------------+------+---------+-------+----------------------------+
  ```
 * 在控制节点配置cinder,创建`GlusterFS`类型:
   
   ```
   cinder type-create GlusterFS
   ```
* 关联类型：
  
  ```
  cinder type-key GlusterFS set volume_backend_name=GlusterFS-Storage
  ```

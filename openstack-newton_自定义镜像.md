###OpenStack-Newton 自定义镜像
官方提供的操作中帮我们指定了一个小镜像，在实际的生产环境中，我们可以自定义所需要的镜像，可以是Windows,Linux和其他常见的操作系统，我们可以对镜像初始的软件环境进行自定义，创建各种不同应用场景的镜像模板。
可参考官方文档： http://docs.openstack.org/image-guide/centos-image.html 

####手动创建CentOS虚拟机
1. 在计算节点上已经安装了我们需要的虚拟机环境（qemu-kvm, libvirt,virt-install）等工具，如果不是利用计算节点来自定义镜像，可以安装以下工具：
 
   ```
   yum install -y qemu-kvm  libvirt virt-install
   ```
   
* 启动libvirt:
  
  ```
  systemctl  start libvirtd
  ```
* 创建虚拟机硬盘,可以根据需求设置大小：
  
  ```
  qemu-img  create -f qcow2 /tmp/centos.qcow2 6G
  
  ```
* 上传一个CentOS的镜像到 tmp目录，创建虚拟机：
  
  ```
  virt-install  --virt-type kvm --name centos --ram 1024 \
  --disk /tmp/centos.qcow2,format=qcow2 --network network=default \
  --graphics vnc,listen=0.0.0.0 --noautoconsole  --os-type=linux \
  --os-variant=rhel7 --location=/tmp/CentOS-7-x86_64-Minimal-1511.iso
  ```
  * 如果使用的是桥接网卡，可以使用：
    
    ```
    virt-install  --virt-type kvm --name centos --ram 1024 \
    --disk /tmp/centos.qcow2,format=qcow2 --network bridge=brqba798366-8f \
    --graphics vnc,listen=0.0.0.0 --noautoconsole  --os-type=linux \
    --os-variant=rhel7 --location=/tmp/CentOS-7-x86_64-Minimal-1511.iso
    ```
* 执行此命令之后可以使用VNC客户端连接到主机上，进行图形化界面的安装操作，在自定义的镜像中，默认使用一个 / 根分区即可。对于分区的格式，默认是lvm,如果后期需要对系统进行容量扩展可以采取这种格式，也可以使用其他标准格式。
* 启动虚拟机,使用vnc再次连接：
  
  ```
  # virsh  list --all
  Id    Name                           State

       centos                         shut off
  
  # virsh start centos
 ```

* 配置初始化环境，网络、主机名，yum源，软件包等，这样一个虚拟机的镜像模板就做好了。

####编写配置脚本启动初始化
在虚拟机启动时，需要使用metadata去做一些初始化的工作，如获取控制节点的ssh public key，修改主机名，初始化网络设置为静态IP地址等。
实现此功能由两种方式：

 * 使用cloud-init获取public key。
 * 在镜像中自定义脚本，通过脚本获取public key 并初始化配置。

这里使用脚本来完成初始化配置。

1. 修改官方提供的获取public key 脚本：

   
   ```
   #!/bin/bash

   set_key(){
    if [ ! -d /root/.ssh ]; then
 	 	mkdir -p /root/.ssh
  		chmod 700 /root/.ssh
  	fi
	for ((i=1;i<=5;i++));do
     	if [ ! -f /root/.ssh/authorized_keys ];then
  		curl -f http://169.254.169.254/latest/meta-data/public-keys/0/openssh-key > /tmp/metadata-key 2>/dev/null
  		if [ $? -eq 0 ];then
    		cat /tmp/metadata-key >> /root/.ssh/authorized_keys
    		chmod 0600 /root/.ssh/authorized_keys
    		restorecon /root/.ssh/authorized_keys
    		rm -f /tmp/metadata-key
    		echo "Successfully retrieved public key from instance metadata"
    		echo "*****************"
    		echo "AUTHORIZED KEYS"
    		echo "*****************"
    		cat /root/.ssh/authorized_keys
    		echo "*****************"
  		fi
    	fi
	done
	}

	set_hostname(){
    	PRE_HOSTNAME=$(curl -s http://169.254.169.254/latest/meta-data/hostname)
    	DOMAIN_NAME=$(echo $PRE_HOSTNAME | awk -F '.' '{print $1}')
    	hostnamectl set-hostname `echo ${DOMAIN_NAME}.example.com`
	}

	set_static_ip(){
    	PRE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
    	NET_FILE="/etc/sysconfig/network-scripts/ifcfg-eth0"
	echo  "TYPE=Ethernet" > $NET_FILE
	echo  "BOOTPROTO=static" >> $NET_FILE
	echo  "NAME=eth0" >> $NET_FILE
	echo  "DEVICE=eth0" >> $NET_FILE
	echo  "ONBOOT=yes" >> $NET_FILE
	echo  "IPADDR=${PRE_IP}" >> $NET_FILE
	echo  "NETMASK=255.255.255.0" >> $NET_FILE
	echo  "GATEWAY=10.0.0.1" >> $NET_FILE
	}

	main(){
   		set_key;
   		set_hostname;
   		set_static_ip;
   		/bin/cp /tmp/rc.local /etc/rc.d/rc.local
   		rm -f /tmp/rc.local
   		reboot
 	}

   	main
    ```

* 此脚本定义了修改的主机名和IP，需要将此脚本放在镜像中，并在镜像rc.local添加脚本自启动：
  
  ```
  echo "/bin/bash /tmp/init.sh" >> /etc/rc.local
  cp /etc/rc.d/rc.local /tmp/
  ```
  
  
  > 提示：`/etc/rc.local`是一个软链接，所指向的文件`/etc/rc.d/rc.local`默认没有执行权限，此处需要添加执行权限才可生效。
  
  ```
  # ls -l /etc/rc.local 
  lrwxrwxrwx. 1 root root 13 Jan 17 03:40 /etc/rc.local -> rc.d/rc.local
  # ls -l /etc/rc.d/rc.local 
  -rw-r--r--. 1 root root 473 Nov 19  2015 /etc/rc.d/rc.local
  # chmod  +x /etc/rc.d/rc.local 
  ```
 
* 关闭镜像系统，将镜像文件`centos.qcow2`上传到控制节点tmp目录，并添加此镜像（与添加cirros镜像命令相同）：
  
  ```
  [root@node1 ~]# source admin-openstack 
[root@node1 ~]# openstack image create "CentOS-7.2-x86_64" \
--file /tmp/centos.qcow2 --disk-format qcow2 --container-format bare --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | db1c5f36acd332fe84c58febbb820365                     |
| container_format | bare                                                 |
| created_at       | 2017-01-17T11:23:46Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/aeb8ffc3-dce6-4e5a-ab9a-bf67a87b9cc2/file |
| id               | aeb8ffc3-dce6-4e5a-ab9a-bf67a87b9cc2                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | CentOS-7.2-x86_64                                    |
| owner            | c34d22ac34cb4c62a025295cd3d6540c                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 1092485120                                           |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2017-01-17T11:24:34Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
  ```
  
* 使用admin 登录Horizon,创建一个能运行此镜像的主机类型，然后回到demo用户，创建`CentOS-7.2-x86_64`类型的云主机。
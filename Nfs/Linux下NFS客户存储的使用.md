## Linux下NFS客户存储的使用

##### 作者: 流雨声

根据之前搭建好的nfs服务器进行配置和使用方式在k8s中的调用。

### 客户端配置

安装nfs-utils客户端

    [root@bogon ~]# yum -y install nfs-utils
    已安装:
      nfs-utils.x86_64 1:1.2.3-70.el6_8.2  
    
    作为依赖被安装:
      keyutils.x86_64 0:1.4-5.el6 libevent.x86_64 0:1.4.13-4.el6 libgssglue.x86_64 0:0.1-11.el6   
      libtirpc.x86_64 0:0.2.1-11.el6_8nfs-utils-lib.x86_64 0:1.1.5-11.el6python-argparse.noarch 0:1.2.1-2.1.el6   
      rpcbind.x86_64 0:0.2.0-12.el6  
    
    完毕！

创建挂载目录

    [root@bogon ~]# mkdir /mnt

查看服务器抛出的共享目录信息

    [root@bogon ~]# showmount -e 192.168.131.20
    Export list for 192.168.131.20:
    /data/lys 192.168.131.0/24

了提高NFS的稳定性，使用TCP协议挂载，NFS默认用UDP协议

    [root@bogon ~]# mount -t nfs 192.168.131.20:/data/lys /mnt -o proto=tcp -o nolock

查看挂载结果
    [root@bogon ~]# df -h


### K8S 使用中部署Redis使用的pv和pvc实例

在nfs或者其他类型后端存储创建pv，首先创建共享目录（在nfs服务器上进行操作）

    [root@nfs ~]# cat /etc/exports
    /k8s/redis-sentinel/0 *(rw,sync,no_subtree_check,no_root_squash)
    /k8s/redis-sentinel/1 *(rw,sync,no_subtree_check,no_root_squash)
    /k8s/redis-sentinel/2 *(rw,sync,no_subtree_check,no_root_squash)


配置生效：

    $exportfs -r
	[root@bogon lys]# service rpcbind start
	[root@bogon lys]# service nfs start


创建pv，注意Redis的空间大小按需修改


    [root@k8s-master01 redis-sentinel]# kubectl create -f redis-sentinel-pv.yaml
    [root@k8s-master01 redis-sentinel]# kubectl get pv | grep redis
    pv-redis-sentinel-0   4GiRWXRecycle  Bound public-service/redis-sentinel-master-storage-redis-sentinel-master-ss-0   redis-sentinel-storage-class 16h
    pv-redis-sentinel-1   4GiRWXRecycle  Bound public-service/redis-sentinel-slave-storage-redis-sentinel-slave-ss-0 redis-sentinel-storage-class 16h
    pv-redis-sentinel-2   4GiRWXRecycle  Bound public-service/redis-sentinel-slave-storage-redis-sentinel-slave-ss-1 redis-sentinel-storage-class 16h


pv文件解析(redis-sentinel-pv.yaml):

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-redis-sentinel-0   #Pv块名字，可以随便定义的
    spec:
      capacity:
    	storage: 4Gi              #存储容量大小
      accessModes:
    	- ReadWriteMany			  #读写模式
      volumeMode: Filesystem
      persistentVolumeReclaimPolicy: Recycle   #是否可以进行回收
      storageClassName: "redis-sentinel-storage-class"  #正常启动的pod会根据存储类名进行pv块的选择。
      nfs:
	    # real share directory
	    path: /k8s/redis-sentinel/0     #nfs服务器存储路径
	    # nfs real ip
	    server: 192.168.131.20				#nfs服务器ip地址


创建pv,创建满足要求的pv存储快:

    $kubectl create -f redis-sentinel-pv.yaml


在创建pod的configmap中使用创建好的pv存储块：

      volumeClaimTemplates:
      - metadata:
          name: redis-sentinel-master-storage  #此处命名随意点，没关系
	    spec:
	      accessModes:
	      - ReadWriteMany                      #和pv创建的读写模式一致
	      storageClassName: "redis-sentinel-storage-class"  #创建的存储类型名
	      resources:
		    requests:
		      storage: 4Gi								#选择已经存在的满足匹配条件的存储块进行存储

至此，我们已经学会了搭建nfs服务器还有对nfs存储进行使用的方式，进行合理的利用。

通过实战pv和pvc进行k8s集群的存储实战演练！

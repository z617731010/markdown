
<font size=6>**<center> 本地NFS实现持久化存储 </center>**</font>

<font size=5>**目录**</font>

* [一.	搭建NFS服务端](#一搭建nfs服务端)
    * [1.	在master端安装NFS](#1在master端安装nfs)
    * [2.	编辑exports文件，添加从机](#2编辑exports文件添加从机)
    * [3.	让修改过的配置文件生效](#3让修改过的配置文件生效)
    * [4. 启动master的NFS服务](#4-启动master的nfs服务)
        * [1）. 先为rpcbind和nfs做开机启动：](#1-先为rpcbind和nfs做开机启动)
		* [2）. 通过查看service列中是否有nfs服务来确认NFS是否启动](#2-通过查看service列中是否有nfs服务来确认nfs是否启动)
		* [3）. 查看可挂载目录及可连接的IP](#3-查看可挂载目录及可连接的ip)
* [二.	配置NFS客户端](#二配置nfs客户端)
	* [1.	安装nfs，并启动服务。在node节点上安装](#1安装nfs并启动服务-在node节点上安装)
* [三.	配置k8s的mysql持久化存储](#三配置k8s的mysql持久化存储)
	* [1.	创建存储类 StorageClass](#1创建存储类-storageclass)
	* [2.	创建持续卷PV(PersistentVolume)](#2创建持续卷pvpersistentvolume)
	* [3.	创建PersistentVolumeClaims](#3创建persistentvolumeclaims)
	* [4. 创建mysql持久化存储](#4-创建mysql持久化存储)
		* [1）创建卷volumes，指定persistentVolumeClaim。](#1创建卷volumes指定persistentvolumeclaim)
		* [2）创建volumeMounts，指定volumes的名字，并指定容器内要共享的目录](#2创建volumemounts指定volumes的名字并指定容器内要共享的目录)

### 一.	搭建NFS服务端
#### 1.	在master端安装NFS
    yum install nfs-utils
    （实际上需要安装两个包nfs-utils和rpcbind, 不过当使用yum安装nfs-utils时会把rpcbind一起安装上）
#### 2.	编辑exports文件，添加从机

    vi /etc/exports
    
    在文件中加入要共享的目录，允许访问的主机
    /data/mysql_db/ 192.168.1.0/24(rw,sync,no_root_squash,fsid=0)
    
    配置说明：
    这一行分为三个部分：
    第一部分：data/mysql_db/ ，这个是本地要共享出去的目录。
    第二部分：192.168.1.0/24 ，允许访问的主机，可以是一个IP：192.168.1.134，也可以是一个IP段：192.168.1.0/24
    第三部分：括号中部分。
    rw表示可读写，ro只读；
    sync ：同步模式，内存中数据时时写入磁盘；async ：不同步，把内存中数据定期写入磁盘中；
    no_root_squash ：加上这个选项后，root用户就会对共享的目录拥有至高的权限控制，就像是对本机的目录操作一样。不安全，不建议使用；root_squash：和上面的选项对应，root用户对共享目录的权限不高，只有普通用户的权限，即限制了root；all_squash：不管使用NFS的用户是谁，他的身份都会被限定成为一个指定的普通用户身份；
    anonuid/anongid ：要和root_squash 以及all_squash一同使用，用于指定使用NFS的用户限定后的uid和gid，前提是本机的/etc/passwd中存在这个uid和gid。
    fsid=0表示将/home/nfs整个目录包装成根目录

#### 3.	让修改过的配置文件生效
    exportfs -arv
    使用exportfs命令，当改变/etc/exports配置文件后，不用重启nfs服务直接用这个exportfs即可，它的常用选项为[-aruv].     
    -a ：全部挂载或者卸载；      
    -r ：重新挂载；      
    -u ：卸载某一个目录；      
    -v ：显示共享的目录；
#### 4. 启动master的NFS服务
##### 1）. 先为rpcbind和nfs做开机启动：    
    systemctl enable rpcbind.service    
    systemctl enable nfs-server.service    
**然后分别启动rpcbind和nfs服务：** 
```
systemctl start rpcbind.service    
systemctl start nfs-server.service    
```
##### 2）. 通过查看service列中是否有nfs服务来确认NFS是否启动
确认NFS服务器启动成功： 

    rpcinfo -p   
        100003    3   tcp   2049  nfs
        100003    4   tcp   2049  nfs
        100227    3   tcp   2049  nfs_acl
        100003    3   udp   2049  nfs
        100003    4   udp   2049  nfs
        100227    3   udp   2049  nfs_acl 

##### 3）. 查看可挂载目录及可连接的IP
    [root@master k8s]# showmount -e 192.168.1.136
    Export list for 192.168.1.136:
    /nfsfileshare  *
    /data/mysql_db 192.168.1.0/24
### 二.	配置NFS客户端
#### 1.	安装nfs，并启动服务。在node节点上安装
    yum install -y nfs-utils
    systemctl enable rpcbind.service
    systemctl start rpcbind.service
    客户端不需要启动nfs服务，只需要启动rpcbind服务.
### 三.	配置k8s的mysql持久化存储
**存储卷的相关文档**
https://kubernetes.io/docs/concepts/storage/volumes/
#### 1.	创建存储类 StorageClass
StorageClass包含的字段provisioner和parameters，当它们用于PersistentVolume属于类需要被动态配置。
StorageClass对象的名称很重要，用户可以如何请求特定的类。
只有同一个类的PV和PVC才能绑定在一起

    [root@master mysql]# cat nfs-class.yaml 
    kind: StorageClass
    apiVersion: storage.k8s.io/v1beta1
    metadata:
      name: nfs-mysql-class
    provisioner: example.com/nfs

#### 2.	创建持续卷PV(PersistentVolume)
PersistentVolume（PV）是集群之中的一块网络存储。跟 Node 一样，也是集群的资源。PV 跟 Volume (卷) 类似，不过会有独立于 Pod 的生命周期。比如一个NFS的PV可以定义为:

    [root@master mysql]# cat pv.yaml 
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: nfs-mysql-pv
    spec:
      capacity:
        storage: 1Mi
      accessModes:
        - ReadWriteMany
      storageClassName: nfs-mysql-class
      nfs:
        # FIXME: use the right IP
        server: 192.168.1.136
    path: "/data/mysql_db"

**创建持续卷**


- accessModes 多节点读写状态：- ReadWriteMany (访问模式)
- 选择持续卷插件为：nfs
- server: 192.168.1.136（NFS服务器IP）
    path: "/data/mysql_db"   (NFS服务器共享目录)

#### 3.	创建PersistentVolumeClaims
PV是存储资源，而PersistentVolumeClaim (PVC) 是对PV的请求。PVC跟Pod类似：
Pod消费Node的源，而PVC消费PV资源；Pod能够请求CPU和内存资源，而PVC请特
定大小和访问模式的数据卷。

    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: nfs-mysql-pvc
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 1Mi
      storageClassName: nfs-mysql-class

这里storage需要PV的 storage相匹配，
并且需要制定和PV相同的存储类  storageClassName: nfs-mysql-class


#### 4. 创建mysql持久化存储
    [root@master mysql]# cat mysql.yaml 
    apiVersion: v1
    kind: Service
    metadata:
      name: mysql
      labels:
        run: mysql
    spec:
      selector:
        run: mysql
      type: NodePort
      ports:
      - port: 3306
        targetPort: 3306
        nodePort: 31006
          
    ---
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: mysql
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            run: mysql
            module: backend
        spec:
          volumes:
          - name: nfs
            persistentVolumeClaim:
              claimName: nfs-mysql-pvc
          containers:
          - name: mysql
            image: mysql:5.7
            env:
            - name: MYSQL_ROOT_PASSWORD
              value: mysql
            - name: EXPLICIT_DEFAULTS_FOR_TIMESTAMP
              value: "true"
            ports:
            - containerPort: 3306
            volumeMounts:
                # name must match the volume name below
                - name: nfs
                  mountPath: "/var/lib/mysql"

##### 1）创建卷volumes，指定persistentVolumeClaim。
      volumes:
      - name: nfs
        persistentVolumeClaim:
          claimName: nfs-mysql-pvc
##### 2）创建volumeMounts，指定volumes的名字，并指定容器内要共享的目录
        volumeMounts:
            # name must match the volume name below
            - name: nfs
              mountPath: "/var/lib/mysql


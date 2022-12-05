# 概览
在之前的文章【[记一次Kubernetes实践](https://mp.weixin.qq.com/s?__biz=Mzg2OTQzMzk3OQ==&mid=2247484534&idx=1&sn=e08fba77e65a32f50023ea73d60764ae&chksm=ce9c5362f9ebda74af9287ba19315b83853a9847b85e5d03704cc37577228e1b977423631aa5&token=1698766384&lang=zh_CN#rd)】和【[记一次K8S下部署Prometheus和Grafana监控Web应用实践](https://mp.weixin.qq.com/s?__biz=Mzg2OTQzMzk3OQ==&mid=2247484572&idx=1&sn=835f2fbb2487322a9e730b58f4c33d3f&chksm=ce9c5388f9ebda9ee36662241d4b762a42b2b17dba49cbfccd134796a53f2db277db5e617fa6&token=1698766384&lang=zh_CN#rd)】中使用的示例应用都没有考虑软件的状态保存，这里状态是指软件在运行过程中的数据存储。
举个例子来说，MySQL就是一个有状态的应用，用户保存的数据不应该在动态扩展或者收缩 `Pod` 资源的时候，把用户数据删除了，因此需要一个能持久化 `Pods` 中数据的方案，这里使用的是Kubernetes中的 `StatefulSet` 资源类。
在使用 `StatefulSet` 资源类之前，需要准备好持久化存储的卷，也就是把数据准备存到哪里。对于Kubernetes而言，是支持三大类卷的，一种是各大云厂商提供的存储，例如awsElasticBlockStore、azureDisk、gcePersistentDisk等；一种是本地化存储盘，例如emptyDir、hostPath等；还有一种是系统管理员或者使用者自己搭建的内部网络存储设备，例如NFS、glusterfs等。
在浏览存储卷的选择上，就好比刘姥姥进大观园，每一种存储都有每一种的特色，都有各自适用的场合，都有各自的局限和长处。然而时过境迁，随着Kubernetes的版本迭代和全球互联网云技术的更新发展，Kubernetes所支持的卷类型也在增增减减。截至目前，开源版的Kubernetes对各大云厂商的卷类型支持度也正在逐步减弱，从v1.25版本来看，AWS的存储方案和Azure的存储方案都不再提供支持了，进入了已弃用的阶段。
然而不管怎么换来换去，有一些存储卷类型生命力是顽强的，比如NFS存储，NFS优点：
>`nfs` 卷能将 NFS (网络文件系统) 挂载到你的 Pod 中。 不像 `emptyDir` 那样会在删除 Pod 的同时也会被删除，`nfs` 卷的内容在删除 Pod 时会被保存，卷只是被卸载。 这意味着 `nfs` 卷可以被预先填充数据，并且这些数据可以在 Pod 之间共享。

所以这里就准备使用NFS存储来进行下一步的工作。
软件环境：
1. Ubuntu：22.04
2. Kubernetes：1.19.0
3. Docker：20.10.21
# 准备NFS
## 安装软件
首先要准备好当作NFS的目录，这里使用的是 `/data/share` 目录。
```bash
$ sudo mkdir /data/share
$ sudo vim /etc/exports
# 添加下面这一行到文档末尾
/data/share   *(rw,sync,no_subtree_check,no_root_squash)
```
准备好目录和配置之后，安装所需要的软件：
```bash
# 安装 rpc-bind
$ sudo apt install rpcbind
# 安装 nfs-kernel-server
$ sudo apt install nfs-kernel-server
```
设置开机自启、启动软件并检查状态
```bash
$ sudo systemctl enable rpcbind && sudo systemctl restart rpcbind
$ sudo systemctl enable nfs-kernel-server && sudo systemctl restart nfs-kernel-server
$ sudo showmount -e localhost
# 输出下面内容即本机配置完成
Export list for localhost:
/data/share *
```
## 确保可用
这时去第二台机器上检查一下是否能挂载该目录：
```bash
# 先安装必要软件
$ sudo apt install nfs-common
$ sudo mkdir /nfs
$ sudo mount 192.168.3.133:/data/share /nfs
$ sudo touch /nfs/hello.txt
# 去主服务服务器上看有没有该文件
$ ls /data/share
hello.txt # 搞定😊
```
# Provisioner，StorageClass，PV，PVC
## 创建操作权限账户和工作空间
这里进行约定，所有有关NFS存储相关的组件，都放到了 `nfs-system` 命名空间下，方便归档和检查。
创建操作权限账户：
```yaml
# nfs-rbac.yaml
apiVersion: v1  
kind: ServiceAccount  
metadata:  
  name: nfs-client-provisioner  
  namespace: nfs-system  
---  
kind: ClusterRole  
apiVersion: rbac.authorization.k8s.io/v1  
metadata:  
  name: nfs-client-provisioner-runner  
rules:  
  - apiGroups: [""]  
    resources: ["persistentvolumes"]  
    verbs: ["get", "list", "watch", "create", "delete"]  
  - apiGroups: [""]  
    resources: ["persistentvolumeclaims"]  
    verbs: ["get", "list", "watch", "update"]  
  - apiGroups: ["storage.k8s.io"]  
    resources: ["storageclasses"]  
    verbs: ["get", "list", "watch"]  
  - apiGroups: [""]  
    resources: ["events"]  
    verbs: ["create", "update", "patch"]  
---  
kind: ClusterRoleBinding  
apiVersion: rbac.authorization.k8s.io/v1  
metadata:  
  name: run-nfs-client-provisioner  
subjects:  
  - kind: ServiceAccount  
    name: nfs-client-provisioner  
    namespace: nfs-system  
roleRef:  
  kind: ClusterRole  
  name: nfs-client-provisioner-runner  
  apiGroup: rbac.authorization.k8s.io  
---  
kind: Role  
apiVersion: rbac.authorization.k8s.io/v1  
metadata:  
  name: leader-locking-nfs-client-provisioner  
  namespace: nfs-system  
rules:  
  - apiGroups: [""]  
    resources: ["endpoints"]  
    verbs: ["get", "list", "watch", "create", "update", "patch"]  
---  
kind: RoleBinding  
apiVersion: rbac.authorization.k8s.io/v1  
metadata:  
  name: leader-locking-nfs-client-provisioner  
  namespace: nfs-system  
subjects:  
  - kind: ServiceAccount  
    name: nfs-client-provisioner  
    namespace: nfs-system  
roleRef:  
  kind: Role  
  name: leader-locking-nfs-client-provisioner  
  apiGroup: rbac.authorization.k8s.io
```
将该资源进行创建：
```bash
$ kubectl create namespace nfs-system
$ kubectl create -f nfs-rbac.yaml -n nfs-system
```
这步之后的所有操作（不包括测试操作），都会在 `nfs-system` 资源下进行操作了。
## Provisioner 建立
`Provisioner` 的意义，就是作为从NFS接入到Kubernetes的一个中间层组件，给Kubernetes提供动态配置持久化卷的能力。
>**NFS subdir external provisioner** is an automatic provisioner that use your _existing and already configured_ NFS server to support dynamic provisioning of Kubernetes Persistent Volumes via Persistent Volume Claims. Persistent volumes are provisioned as `${namespace}-${pvcName}-${pvName}`..

这里使用的是应用广泛的 `nfs-subdir-external-provisioner` 组件，Github地址：[nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)，官方提供的配置文件中，有一些需要从Google的容器仓库中拉取镜像，然而这是行不通的，所以我把它们放到了我的镜像仓库中，方便使用。
需要注意的是，该配置必须满足Kubernetes版本在v1.19及以上。
具体的查看配置文件：
```yaml
# nfs-provisioner-deployment.yaml
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: nfs-client-provisioner  
  labels:  
    app: nfs-client-provisioner  
spec:  
  replicas: 1  
  strategy:  
    type: Recreate  # 当更新该组件时，采取什么策略
  selector:  
    matchLabels:  
      app: nfs-client-provisioner  
  template:  
    metadata:  
      labels:  
        app: nfs-client-provisioner  
    spec:  
      serviceAccountName: nfs-client-provisioner  
      containers:  
        - name: nfs-client-provisioner  
          image: registry.cn-hangzhou.aliyuncs.com/pub-repo/nfs-subdir-external-provisioner  
          imagePullPolicy: IfNotPresent  
          volumeMounts:  
            - name: nfs-client-root  
              mountPath: /persistentvolumes  
          env:  
            - name: PROVISIONER_NAME     # 在该处设置的 provisioner 将要用在 StorageClass 中  
              value: nfs-client  
            - name: NFS_SERVER  
              value: 192.168.3.133  
            - name: NFS_PATH  
              value: /data/share  
      volumes:  
        - name: nfs-client-root  
          nfs:  
            server: 192.168.3.133      
            path: /data/share            
```
以上配置详情可以从官方资源文件下的template文件夹下找到初始模板，根据需要可以进一步配置，这里仅作实验性配置，所以少了很多东西，但能用😉。
创建好后，放到Kubernetes集群中创建对应资源：
```bash
$ kubectl create -f ./nfs-provisioner-deploy.yaml -n nfs-system
$ kubectl get deployment -n nfs-system
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
nfs-client-provisioner   1/1     1            1           2d5h
```
## StorageClass资源创建
```yaml
# nfs-storageclass.yaml
apiVersion: storage.k8s.io/v1  
kind: StorageClass  
metadata:  
  name: nfs-ssd  
  annotations:  
    storageclass.kubernetes.io/is-default-class: "false"   
provisioner: nfs-client    # 和上文中创建的PROVISIONER_NAME字段值一致                                
parameters:  
  archiveOnDelete: "true"  # 删除PVC保留数据，改为"false"则不保留
mountOptions:  
  - hard                                                  
  - nfsvers=4              # NFS版本
```
创建并检查是否正常：
```bash
$ kubectl create -f ./nfs-storageclass.yaml
$ kubectl get storageclass
NAME      PROVISIONER   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-ssd   nfs-client    Delete          Immediate           false                  27h
```
可以看到这里和 `provisioner` 进行了绑定，并且创建了名称为 `nfs-ssd` 的存储类型。
## PV 资源创建
这里先创建一个用于测试用的Persistent Volume，来确保一切可用：
```yaml
# nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    app: nfs-pv-test
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs-ssd
  mountOptions:
    - hard
    - nfsvers=4
  nfs:
    path: /data/share
    server: 192.168.3.133
```
然后创建资源，并检查状态：
```bash
$ kubectl create -f ./nfs-pv.yaml
$ kubectl get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                        STORAGECLASS   REASON   AGE
nfs-pv              1Gi        RWO            Recycle          Available                                nfs-ssd                 4s
```
可以看到状态已经是 `Available` 了。
## PVC资源创建
```yaml
# nfs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 3Mi
  storageClassName: nfs-ssd
  selector:
    matchLabels:
      app: nfs-pv-test
```
创建资源并检查状态：
```bash
$ kubectl create -f ./nfs-pvc.yaml
$ kubectl get pvc
NAME                 STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc              Bound    nfs-pv              1Gi        RWO            nfs-ssd        3s
$ kubectl get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   REASON   AGE
nfs-pv              1Gi        RWO            Recycle          Bound    default/nfs-pvc              nfs-ssd                 3m14s
```
此时可以看到 `PV` 状态变成了 `Bound`，也就是绑定到了 PVC 上，一切正常😉。
## 测试存储是否正常使用
这里使用 busybox 容器进行测试：
```yaml
# nfs-test-all.yaml
kind: Pod
apiVersion: v1
metadata:
  name: nfs-test-all
spec:
  containers:
    - name: nfs-test-all
      image: busybox:latest
      command:
        - "/bin/sh"
      args:
        - "-c"
        - "touch /nfs/hello.txt"  # 在 nfs 目录下创建hello.txt文件
      volumeMounts:
        - name: nfs-pvc
          mountPath: "/nfs" # 挂载 nfs 目录到根目录下
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: nfs-pvc
```
创建该Pod，并检查是否在NFS服务器上是否有创建的对应的文件。
```bash
$ kubectl create -f ./nfs-test-all.yaml
$ kubectl get pods
NAME           READY   STATUS      RESTARTS   AGE
nfs-test-all   0/1     Completed   0          13s
$ ll /data/share
-rw-r--r-- 1 root root    0 12月  4 17:19 hello.txt
```
可以看到确实创建了对应的文件，所以到目前这一步，还是一切按计划进行着。
# StatefulSet
## 创建StatefulSet资源MySQL
接下来创建的资源就要和具体的应用相关了，由于这里使用的是MySQL数据库进行的持久化存储测试，所以创建一个给MySQL数据库使用的Persistent Volume，并且创建必要的 `StorageClass`、`ConfigMap`、`StatefulSet` 和 `Service` 来进行验证：
```yaml
# mysql-all-in-one.yaml
apiVersion: storage.k8s.io/v1  
kind: StorageClass  
metadata:  
  name: nfs-ssd  
  annotations:  
    storageclass.kubernetes.io/is-default-class: "false"  
provisioner: nfs-client  
parameters:  
  archiveOnDelete: "true"  
mountOptions:  
  - hard  
  - nfsvers=4  
  
---  
# MySQL配置，主要配置的是密码  
apiVersion: v1  
kind: ConfigMap  
metadata:  
  name: mysql-config-map  
data:  
  MYSQL_ROOT_PASSWORD: '666666'  
  
---  
  
# 数据存储卷  
apiVersion: v1  
kind: PersistentVolume  
metadata:  
  name: nfs-pv-data-mysql  
spec:  
  capacity:  
    storage: 1Gi  
  volumeMode: Filesystem  
  accessModes:  
    - ReadWriteOnce  
  persistentVolumeReclaimPolicy: Retain  
  storageClassName: nfs-ssd  
  nfs:  
    server: 192.168.3.133  
    path: /data/share/mysql  
  
---  
# MySQL有状态资源  
apiVersion: apps/v1  
kind: StatefulSet  
metadata:  
  name: mysql  
spec:  
  replicas: 1  
  serviceName: mysql-service  
  selector:  
    matchLabels:  
      app: mysql  
  template:  
    metadata:  
      labels:  
        app: mysql  
    spec:  
      containers:  
        - name: mysql  
          image: mysql:5.7  
          imagePullPolicy: IfNotPresent  
          ports:  
            - name: mysql  
              containerPort: 3306  
              protocol: TCP  
          envFrom:  
            - configMapRef:  
                name: mysql-config-map  
          resources:  
            requests:  
              memory: 512Mi  
            limits:  
              memory: 512Mi  
          volumeMounts:  
            - name: mysql-data  
              mountPath: /var/lib/mysql  
      imagePullSecrets:  
        - name: k8s-auth  
  volumeClaimTemplates:  
    - metadata:  
        name: mysql-data  
      spec:  
        accessModes: ["ReadWriteOnce"]  
        volumeMode: Filesystem  
        resources:  
          requests:  
            storage: 1Gi  
        storageClassName: nfs-ssd  
  
---  
# 内部访问的headless服务  
apiVersion: v1  
kind: Service  
metadata:  
  name: mysql-service  
spec:  
  selector:  
    app: mysql  
  ports:  
    - name: mysql  
      port: 3306  
  clusterIP: None  
  
---  
# 外部使用的服务  
apiVersion: v1  
kind: Service  
metadata:  
  name: mysql-external-service  
spec:  
  selector:  
    app: mysql  
  ports:  
    - name: mysql  
      protocol: TCP  
      port: 3306  
      targetPort: 3306  
      nodePort: 30306  
  type: NodePort
```
配置完成后，创建该资源：
```bash
$ kubectl create -f ./mysql-all-in-one.yaml 
storageclass.storage.k8s.io/nfs-ssd created
configmap/mysql-config-map created
persistentvolume/nfs-pv-data-mysql created
statefulset.apps/mysql created
service/mysql-service created
service/mysql-external-service created
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   1/1     Running   0          11s
$ ll /data/share/mysql
总用量 188488
drwxr-xr-x 5  999 root       4096 12月  4 17:47 ./
drwxr-xr-x 3 root root       4096 12月  4 17:45 ../
-rw-r----- 1  999 docker       56 12月  4 17:46 auto.cnf
-rw------- 1  999 docker     1680 12月  4 17:46 ca-key.pem
-rw-r--r-- 1  999 docker     1112 12月  4 17:46 ca.pem
-rw-r--r-- 1  999 docker     1112 12月  4 17:46 client-cert.pem
-rw------- 1  999 docker     1676 12月  4 17:46 client-key.pem
-rw-r----- 1  999 docker     1310 12月  4 17:47 ib_buffer_pool
-rw-r----- 1  999 docker 79691776 12月  4 17:47 ibdata1
-rw-r----- 1  999 docker 50331648 12月  4 17:47 ib_logfile0
-rw-r----- 1  999 docker 50331648 12月  4 17:46 ib_logfile1
-rw-r----- 1  999 docker 12582912 12月  4 17:47 ibtmp1
drwxr-x--- 2  999 docker     4096 12月  4 17:46 mysql/
lrwxrwxrwx 1  999 docker       27 12月  4 17:46 mysql.sock -> /var/run/mysqld/mysqld.sock
drwxr-x--- 2  999 docker     4096 12月  4 17:46 performance_schema/
-rw------- 1  999 docker     1676 12月  4 17:46 private_key.pem
-rw-r--r-- 1  999 docker      452 12月  4 17:46 public_key.pem
-rw-r--r-- 1  999 docker     1112 12月  4 17:46 server-cert.pem
-rw------- 1  999 docker     1676 12月  4 17:46 server-key.pem
drwxr-x--- 2  999 docker    12288 12月  4 17:46 sys/
```
到目前，MySQL是正常启动的，现在尝试使用客户端去连接该服务，并创建一个表。
## 创建表并插入数据
创数据库和表语句以及插入数据语句：
```sql
create database test_mysql;  
  
use test_mysql;  
  
create table msg (  
  line INT(20) primary key auto_increment,  
  msg varchar(255)  
);  
  
insert into msg(line, msg) values (null, 'hello,');  
insert into msg(line, msg) values (null, 'world!');  
  
select msg from msg;
+--------+
| msg    |
+--------+
| hello, |
| world! |
+--------+
2 rows in set (0.00 sec)
```
可以看到该MySQL数据库已经可以正常使用了，接下来看看是否保存在NFS存储卷中，
```bash
$ ls /data/share/mysql
auto.cnf         ib_buffer_pool  mysql               server-cert.pem
ca-key.pem       ibdata1         mysql.sock          server-key.pem
ca.pem           ib_logfile0     performance_schema  sys
client-cert.pem  ib_logfile1     private_key.pem     test_mysql
client-key.pem   ibtmp1          public_key.pem
```
可以看到一切都正常了，接下来验证删除Pod和StatefulSet之后，是否仍然可以保留数据
```bash
$ kubectl get pods -w 
NAME      READY   STATUS    RESTARTS   AGE
mysql-0   1/1     Running   0          107m
mysql-0   1/1     Terminating   0          107m
mysql-0   1/1     Terminating   0          107m
mysql-0   0/1     Terminating   0          107m
mysql-0   0/1     Terminating   0          107m
mysql-0   0/1     Terminating   0          107m
mysql-0   0/1     Pending       0          0s
mysql-0   0/1     Pending       0          0s
mysql-0   0/1     ContainerCreating   0          0s
mysql-0   0/1     ContainerCreating   0          6s
mysql-0   1/1     Running             0          8s
$ kubectl exec -it mysql-0 -- /bin/bash
mysql> use test_mysql;select msg from msg;
Database changed
+--------+
| msg    |
+--------+
| hello, |
| world! |
+--------+
2 rows in set (0.01 sec)
```
可以看到删除了Pod之后，自动重建的Pods还是可以使用原来的数据的，得以保留，那么再直接删除对应的StatefulSet看看：
```bash
$ kubectl delete statefulset mysql
statefulset.apps "mysql" deleted
$ ls /data/share/mysql
auto.cnf         ib_buffer_pool  mysql.sock          server-key.pem
ca-key.pem       ibdata1         performance_schema  sys
ca.pem           ib_logfile0     private_key.pem     test_mysql
client-cert.pem  ib_logfile1     public_key.pem
client-key.pem   mysql           server-cert.pem
```
可以看到卷得以保存，数据没有丢失，这时候再创建同样的StatefulSet，并挂载同样的PV资源，看看能否继续使用：
```bash
$ kubectl exec -it mysql-0 -- /bin/bash
mysql> use test_mysql; select * from msg;

Database changed
+------+--------+
| line | msg    |
+------+--------+
|    1 | hello, |
|    2 | world! |
+------+--------+
2 rows in set (0.00 sec)
```
可以看出来仍然是正常数据的，没有丢失。如果把StatefulSet、PVC、PV都删掉的话，仍然是不会删掉NFS文件系统中数据文件的，需要操作系统管理员进行手动删除才会删除干净，或者挂载其他不同的文件目录。
这里其实是有一个需要注意的地方，在 `persistentVolumeReclaimPolicy` 的值设置上，可以选择 `Retain`，也可以选择 `Delete`，也可以选择 `Recycle`，但是 `Recycle` 现在已经进入了废弃阶段，因为他会导致不可预估的数据丢失问题。
# 关于PV，PVC和StorageClass的一个小问题
当PV、PVC都没有声明StorageClass时候，或者两者声明的StorageClass并不存在时候，PV和PVC是可以正常工作的。
当PV和PVC声明的StorageClass不同时，PVC找不到对应的PV，这时PVC会卡在Pending状态，报错找不到对应的存储卷。
当PVC声明了StorageClass，并且对应的StorageClass真的存在并且正常工作时，即便不声明PV资源，PVC也会自动创建PV，然后绑定下来，不过这时PVC是不能使用 `selector` 的，想想也是，都没有可以 `select` 的PV。对于这种自动生成的方式，会根据 `StorageClass` 中设置的 `archiveOnDelete: "true"` 字段决定是否要归档，如果不设置的话，会默认进行删除，如果设置了，会把原来的文件夹设置为 `archived-***-***...` 以表示归档。
>PersistentVolumes 可以有多种回收策略，包括 "Retain"、"Recycle" 和 "Delete"。 对于动态配置的 PersistentVolumes 来说，默认回收策略为 "Delete"。 这表示当用户删除对应的 PersistentVolumeClaim 时，动态配置的 volume 将被自动删除。 如果 volume 包含重要数据时，这种自动行为可能是不合适的。 那种情况下，更适合使用 "Retain" 策略。 使用 "Retain" 时，如果用户删除 PersistentVolumeClaim，对应的 PersistentVolume 不会被删除。 相反，它将变为 Released 状态，表示所有的数据可以被手动恢复。

这是来自Google的一段话。

---
快要见面，好不开心！😉






























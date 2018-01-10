# kubernetes-ceph
## kubernetes中使用ceph rbd作为持久化存储
#### 1、创建ceph-secret

创建ceph-secret这个k8s secret对象，这个secret对象用于k8s volume插件访问ceph集群：
获取client.admin的keyring值，并用base64编码：
    # ceph auth get-key client.admin
    AQCBbgFa4tFTDxAA5Y4fvdDL3sntAyLFmnQwpQ==
    
    # echo "AQCBbgFa4tFTDxAA5Y4fvdDL3sntAyLFmnQwpQ=="|base64
    QVFDQmJnRmE0dEZURHhBQTVZNGZ2ZERMM3NudEF5TEZtblF3cFE9PQo=
    
在k8s-cephrbd下建立ceph-secret.yaml文件，data下的key字段值即为上面得到的编码值：
    //ceph-secret.yaml
    
    apiVersion: v1
    kind: Secret
    metadata:
      name: ceph-secret
    data:
      key: QVFDQmJnRmE0dEZURHhBQTVZNGZ2ZERMM3NudEF5TEZtblF3cFE9PQo=
      
创建ceph-secret：
        # kubectl create -f ceph-secret.yaml
        secret "ceph-secret" created
        
        # kubectl get secret
        NAME                  TYPE                                  DATA      AGE
        ceph-secret           Opaque                                1         16s


1、创建disk image

    $ ceph osd pool create rbd 100
    ceph osd pool create rbd 100
    # rbd create mysql-server1 -s 4096
    [root@server1 containers]# rbd info rbd/mysql-server1
    rbd image 'mysql-server1':
            size 4096 MB in 1024 objects
            order 22 (4096 kB objects)
            block_name_prefix: rbd_data.6ee042ae8944a
            format: 2
            features: layering
            flags: 
            create_timestamp: Mon Nov 13 14:58:27 2017
设置ceph元数据算法：

    ceph osd crush tunables legacy
    
关闭mysql-server1中内核功能参数：
    
    rbd feature disable mysql-server1 exclusive-lock, object-map, fast-diff, deep-flatten

如果这里不先创建一个ceph-image，后续Pod启动时，会出现如下的一些错误，比如pod始终处于ContainerCreating状态：

    # kubectl get pod
    NAME                        READY     STATUS              RESTARTS   AGE
    ceph-pod1                   0/1       ContainerCreating   0          13s

如果出现这种错误情况，可以查看/var/log/upstart/kubelet.log，你也许能看到如下错误信息：

    I1107 06:02:27.500247   22037 operation_executor.go:768] MountVolume.SetUp succeeded for volume "kubernetes.io/secret/01d049c6-9430-11e6-ba01-00163e1625a9-default-token-40z0x" (spec.Name: "default-token-40z0x") pod "01d049c6-9430-11e6-ba01-00163e1625a9" (UID: "01d049c6-9430-11e6-ba01-00163e1625a9").
    I1107 06:03:08.499628   22037 reconciler.go:294] MountVolume operation started for volume "kubernetes.io/rbd/ea848a49-a46b-11e6-ba01-00163e1625a9-ceph-pv" (spec.Name: "ceph-pv") to pod "ea848a49-a46b-11e6-ba01-00163e1625a9" (UID: "ea848a49-a46b-11e6-ba01-00163e1625a9").
    E1107 06:03:09.532348   22037 disk_manager.go:56] failed to attach disk
    E1107 06:03:09.532402   22037 rbd.go:228] rbd: failed to setup mount /var/lib/kubelet/pods/ea848a49-a46b-11e6-ba01-00163e1625a9/volumes/kubernetes.io~rbd/ceph-pv rbd: map failed exit status 2 rbd: sysfs write failed
    In some cases useful info is found in syslog - try "dmesg | tail" or so.
    rbd: map failed: (2) No such file or directory

#### 2、创建PV
我们直接复用之前创建的ceph-secret对象，PV的描述文件ceph-pv.yaml如下：

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: ceph-pv-mysql-server1
    spec:
      capacity:
        storage: 4Gi
      accessModes:
        - ReadWriteOnce
      rbd:
        monitors:
          - 192.168.31.150:6789
        pool: rbd
        image: mysql-server1
        user: admin
        secretRef:
          name: ceph-secret
        fsType: ext4
        readOnly: false
      persistentVolumeReclaimPolicy: Recycle
执行创建操作：

    # kubectl create -f ceph-pv.yaml
    persistentvolume "ceph-pv" created

    # kubectl get pv
    [root@server1 server1]# kubectl get pv
    NAME                   CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                          STORAGECLASS   REASON    AGE
    ceph-pv-mysql-server1  4Gi        RWO           Recycle         Bound     default/ceph-pvc-mysql-server1                          58d
    
#### 3、创建PVC

pvc是Pod对Pv的请求，将请求做成一种资源，便于管理以及pod复用。我们用到的pvc描述文件ceph-pvc.yaml如下：

    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: ceph-pvc-mysql-server1
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 4Gi

执行创建操作：

    # kubectl create -f ceph-pvc.yaml
    persistentvolumeclaim "ceph-claim" created

    # kubectl get pvc
    NAME                          STATUS    VOLUME                       CAPACITY   ACCESSMODES   STORAGECLASS   AGE
    ceph-pvc-mysql-server1        Bound     ceph-pv-mysql-server1        4Gi        RWO                          58d

#### 4、创建挂载ceph RBD的pod

pod描述文件ceph-pod1.yaml如下：

    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: mysql-server1
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: mysql-server1
        spec:
          containers:
          - name: mysql-server1
            image: 192.168.31.150:5000/mysql:latest
            volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
              readOnly: false
            env:
            - name: MYSQL_ROOT_PASSWORD
              value: "******"
            ports:
            - containerPort: 3306
          imagePullSecrets:
            - name: myregistrykey
          volumes:
          #- name: mysql-runfile
          #  hostPath:
          #    path: /share/kubernetes/mysql/server1/test
          - name: mysql-data
            persistentVolumeClaim:
              claimName: ceph-pvc-mysql-server1
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: mysql-server1
      labels:
        app: mysql-server1
    spec:
      type: NodePort           
      ports:
      - port: 3306                     #内部接口
      selector:
        app: mysql-server1
创建pod操作：
    # kubectl create -f ceph-pod1.yaml
    pod "ceph-pod1" created
    
    # kubectl get pod
    NAME                        READY     STATUS              RESTARTS   AGE
    mysql-server1                   0/1       ContainerCreating   0          13s

    # kubectl get pod
    [root@server1 server1]# kubectl get pod
    NAME                                     READY     STATUS    RESTARTS   AGE
    mysql-server1-1657545038-4gnrd           1/1       Running   13         49d

查看该镜像使用率

    [root@node2 ~]# df -lh
    文件系统                 容量  已用  可用 已用% 挂载点
    /dev/mapper/centos-root   36G  8.2G   28G   23% /
    devtmpfs                 1.2G     0  1.2G    0% /dev
    tmpfs                    1.2G   84K  1.2G    1% /dev/shm
    tmpfs                    1.2G   89M  1.1G    8% /run
    tmpfs                    1.2G     0  1.2G    0% /sys/fs/cgroup
    /dev/sda1                497M  157M  341M   32% /boot
    /dev/sdb1                 97M  5.4M   92M    6% /var/lib/ceph/osd/ceph-2
    tmpfs                    231M   16K  231M    1% /run/user/42
    /dev/rbd0                3.9G  226M  3.4G    7% /var/lib/kubelet/plugins/kubernetes.io/rbd/rbd/rbd-image-mysql-server1


# kubernetes-ceph
kubernetes中使用ceph rbd作为持久化存储
创建ceph-secret

接下来我们来创建ceph-secret这个k8s secret对象，这个secret对象用于k8s volume插件访问ceph集群：
获取client.admin的keyring值，并用base64编码：
# ceph auth get-key client.admin
AQBiKBxYuPXiJRAAsupnTBsURoWzb0k00oM3iQ==

# echo "AQBiKBxYuPXiJRAAsupnTBsURoWzb0k00oM3iQ=="|base64
QVFCaUtCeFl1UFhpSlJBQXN1cG5UQnNVUm9XemIwazAwb00zaVE9PQo=
在k8s-cephrbd下建立ceph-secret.yaml文件，data下的key字段值即为上面得到的编码值：
//ceph-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
data:
  key: QVFCaUtCeFl1UFhpSlJBQXN1cG5UQnNVUm9XemIwazAwb00zaVE9PQo=
创建ceph-secret：
# kubectl create -f ceph-secret.yaml
secret "ceph-secret" created

# kubectl get secret
NAME                  TYPE                                  DATA      AGE
ceph-secret           Opaque                                1         16s


1、创建disk image

$ rbd create ceph-image -s 128 #考虑后续format快捷，这里只用了128M，仅适用于Demo哦。
ceph osd pool create rbd 100
# rbd create ceph-image -s 128
# rbd info rbd/ceph-image
rbd image 'ceph-image':
    size 128 MB in 32 objects
    order 22 (4096 kB objects)
    block_name_prefix: rbd_data.37202ae8944a
    format: 2
    features: layering
    flags:
配置
ceph osd crush tunables legacy
rbd feature disable ceph-image exclusive-lock, object-map, fast-diff, deep-flatten

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
2、创建PV

我们直接复用之前创建的ceph-secret对象，PV的描述文件ceph-pv.yaml如下：
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
      - 10.47.136.60:6789
    pool: rbd
    image: ceph-image
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
NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     REASON    AGE
ceph-pv   1Gi        RWO           Recycle         Available                       7s
3、创建PVC

pvc是Pod对Pv的请求，将请求做成一种资源，便于管理以及pod复用。我们用到的pvc描述文件ceph-pvc.yaml如下：
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

执行创建操作：
# kubectl create -f ceph-pvc.yaml
persistentvolumeclaim "ceph-claim" created

# kubectl get pvc
NAME         STATUS    VOLUME    CAPACITY   ACCESSMODES   AGE
ceph-claim   Bound     ceph-pv   1Gi        RWO           12s

4、创建挂载ceph RBD的pod

pod描述文件ceph-pod1.yaml如下：
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod1
spec:
  containers:
  - name: ceph-busybox1
    image: busybox
    command: ["sleep", "600000"]
    volumeMounts:
    - name: ceph-vol1
      mountPath: /usr/share/busybox
      readOnly: false
  volumes:
  - name: ceph-vol1
    persistentVolumeClaim:
      claimName: ceph-claim
创建pod操作：
# kubectl create -f ceph-pod1.yaml
pod "ceph-pod1" created

# kubectl get pod
NAME                        READY     STATUS              RESTARTS   AGE
ceph-pod1                   0/1       ContainerCreating   0          13s
Pod还处于ContainerCreating状态。pod的创建，尤其是挂载pv的Pod的创建需要一小段时间，耐心等待一下，我们可以查看一下/var/log/upstart/kubelet.log：
I1107 11:44:38.768541   22037 mount_linux.go:272] `fsck` error fsck from util-linux 2.20.1

fsck.ext2: Bad magic number in super-block while trying to open /dev/rbd1
/dev/rbd1:
The superblock could not be read or does not describe a valid ext2/ext3/ext4
filesystem.  If the device is valid and it really contains an ext2/ext3/ext4
filesystem (and not swap or ufs or something else), then the superblock
is corrupt, and you might try running e2fsck with an alternate superblock:
    e2fsck -b 8193 <device>
 or
    e2fsck -b 32768 <device>

E1107 11:44:38.774080   22037 mount_linux.go:110] Mount failed: exit status 32
Mounting arguments: /dev/rbd1 /var/lib/kubelet/plugins/kubernetes.io/rbd/rbd/rbd-image-ceph-image ext4 [defaults]
Output: mount: wrong fs type, bad option, bad superblock on /dev/rbd1,
       missing codepage or helper program, or other error
       In some cases useful info is found in syslog - try
       dmesg | tail  or so

I1107 11:44:38.839148   22037 mount_linux.go:292] Disk "/dev/rbd1" appears to be unformatted, attempting to format as type: "ext4" with options: [-E lazy_itable_init=0,lazy_journal_init=0 -F /dev/rbd1]
I1107 11:44:39.152689   22037 mount_linux.go:297] Disk successfully formatted (mkfs): ext4 - /dev/rbd1 /var/lib/kubelet/plugins/kubernetes.io/rbd/rbd/rbd-image-ceph-image
I1107 11:44:39.220223   22037 operation_executor.go:768] MountVolume.SetUp succeeded for volume "kubernetes.io/rbd/811a57ee-a49c-11e6-ba01-00163e1625a9-ceph-pv" (spec.Name: "ceph-pv") pod "811a57ee-a49c-11e6-ba01-00163e1625a9" (UID: "811a57ee-a49c-11e6-ba01-00163e1625a9").
可以看到，k8s通过fsck发现这个image是一个空image，没有fs在里面，于是默认采用ext4为其格式化，成功后，再行挂载。等待一会后，我们看到ceph-pod1成功run起来了：
# kubectl get pod
NAME                        READY     STATUS    RESTARTS   AGE
ceph-pod1                   1/1       Running   0          4m

# docker ps
CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS              PORTS               NAMES
f50bb8c31b0f        busybox                                               "sleep 600000"           4 hours ago         Up 4 hours                              k8s_ceph-busybox1.c0c0379f_ceph-pod1_default_811a57ee-a49c-11e6-ba01-00163e1625a9_9d910a29

# docker exec 574b8069e548 df -h
Filesystem                Size      Used Available Use% Mounted on
none                     39.2G     20.9G     16.3G  56% /
tmpfs                     1.9G         0      1.9G   0% /dev
tmpfs                     1.9G         0      1.9G   0% /sys/fs/cgroup
/dev/vda1                39.2G     20.9G     16.3G  56% /dev/termination-log
/dev/vda1                39.2G     20.9G     16.3G  56% /etc/resolv.conf
/dev/vda1                39.2G     20.9G     16.3G  56% /etc/hostname
/dev/vda1                39.2G     20.9G     16.3G  56% /etc/hosts
shm                      64.0M         0     64.0M   0% /dev/shm
/dev/rbd1               120.0M      1.5M    109.5M   1% /usr/share/busybox
tmpfs                     1.9G     12.0K      1.9G   0% /var/run/secrets/kubernetes.io/serviceaccount
tmpfs                     1.9G         0      1.9G   0% /proc/kcore
tmpfs                     1.9G         0      1.9G   0% /proc/timer_list
tmpfs                     1.9G         0      1.9G   0% /proc/timer_stats
tmpfs                     1.9G         0      1.9G   0% /proc/sched_debug
六、简单测试

这一节我们要对cephrbd作为k8s PV的效用做一个简单测试。测试步骤：
1) 在container中，向挂载的cephrbd写入数据；
2) 删除ceph-pod1
3) 重新创建ceph-pod1，查看数据是否还存在。
我们首先通过touch 、vi等命令向ceph-pod1挂载的cephrbd volume写入数据：我们通过容器f50bb8c31b0f 创建/usr/share/busybox/hello-ceph.txt，并向文件写入”hello ceph”一行字符串并保存。
# docker exec -it f50bb8c31b0f touch /usr/share/busybox/hello-ceph.txt
# docker exec -it f50bb8c31b0f vi /usr/share/busybox/hello-ceph.txt
# docker exec -it f50bb8c31b0f cat /usr/share/busybox/hello-ceph.txt
hello ceph
接下来删除ceph-pod1：
# kubectl get pod
NAME                        READY     STATUS    RESTARTS   AGE
ceph-pod1                   1/1       Running   0          4h

# kubectl delete pod/ceph-pod1
pod "ceph-pod1" deleted

# kubectl get pod
NAME                        READY     STATUS        RESTARTS   AGE
ceph-pod1                   1/1       Terminating   0          4h

# kubectl get pv,pvc
NAME         CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                REASON    AGE
pv/ceph-pv   1Gi        RWO           Recycle         Bound     default/ceph-claim             4h
NAME             STATUS    VOLUME    CAPACITY   ACCESSMODES   AGE
pvc/ceph-claim   Bound     ceph-pv   1Gi        RWO           4h

可以看到ceph-pod1的删除需要一段时间，这段时间pod一直处于“ Terminating”状态。同时，我们看到pod的删除并没有影响到pv和pvc object，它们依旧存在。
最后，我们再次来创建一下一个使用同一个pvc的pod，为了避免“不必要”的麻烦，我们建立一个名为ceph-pod2.yaml的描述文件：
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod2
spec:
  containers:
  - name: ceph-busybox2
    image: busybox
    command: ["sleep", "600000"]
    volumeMounts:
    - name: ceph-vol2
      mountPath: /usr/share/busybox
      readOnly: false
  volumes:
  - name: ceph-vol2
    persistentVolumeClaim:
      claimName: ceph-claim
创建ceph-pod2：
# kubectl create -f ceph-pod2.yaml
pod "ceph-pod2" created

root@node1:~/k8stest/k8s-cephrbd# kubectl get pod
NAME                        READY     STATUS    RESTARTS   AGE
ceph-pod2                   1/1       Running   0          14s

root@node1:~/k8stest/k8s-cephrbd# docker ps
CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS              PORTS               NAMES
574b8069e548        busybox                                               "sleep 600000"           11 seconds ago      Up 10 seconds                           k8s_ceph-busybox2.c5e637a1_ceph-pod2_default_f4aeebd6-a4c3-11e6-ba01-00163e1625a9_fc94c0fe
查看数据是否依旧存在：
# docker exec -it 574b8069e548 cat /usr/share/busybox/hello-ceph.txt
hello ceph
数据完好无损的被ceph-pod2读取到了！

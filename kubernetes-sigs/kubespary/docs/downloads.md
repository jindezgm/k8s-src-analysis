# 下载二进制文件和容器

Kubespray支持多种下载/上传模式，默认模式为：

* 每个节点自己下载二进制文件和容器镜像，即`download_run_once: False`;
* 对于Kubernetes应用，拉取镜像策略是`k8s_image_pull_policy: IfNotPresent`。
* 对于系统管理的容器，如kubelet或etcd，拉取镜像策略是`download_always_pull: False`，即只有需要的repo:tag或sha256哈希值与本地不同才拉取镜像。

还有一种“拉一次，推多次”模式：

* 设置`download_run_once: True`，容器镜像和二进制文件kubespray只下载一次，然后将它们推送到集群所有节点。默认委托第一个控制平面节点执行下载。
* 设置`download_localhost: True`不在委托第一个控制平面节点，变为自己下载，这在集群节点无法访问外部地址时非常有用。这需要在Ansible节点上安装并运行容器运行时，并且当前用户要么在docker组中，要么可以执行无密码sudo，以便能够使用容器运行时。注意：即使`download_localhost`为False，文件仍然会从委托的下载节点复制到Ansible节点（本地主机），然后从Ansible节点分发到所有集群节点。

注意：当`download_run_once`为true和`download_localhost`为false时，所有下载都将在委托节点（即第一个控制平面节点）上完成，包括该节点上不需要的容器镜像的下载。因此，该节点上所需的存储可能会比`download_run_once`为false时更多，因为所有镜像都将下载到该节点上容器运行时的存储中，而不仅仅是该节点所需的镜像。

关于缓存：

* 当`download_run_once`是True，所有下载的文件都会缓存在本地`download_cache_dir`，默认是`/tmp/kubespray_cache`。 在随后的下载过程中，此本地缓存将成为节点下载文件和镜像的源，最大限度地减少公网带宽使用并缩短下载时间。预计在ansible节点上将使用大约800MB的磁盘空间用于缓存。kubernetes节点上的镜像缓存所需的磁盘空间与最大镜像所需的磁盘空间差不多，目前略小于150MB。
* 默认情况下，如果`download_run_once`为False，kubespray不会将下载的镜像和文件从委托节点拷贝到本地缓存，也不会使用该缓存作为这些节点的下载源。如果您有容器镜像和文件的完整缓存，并且不需要下载任何内容，此时想要使用缓存，需要设置`download_force_cache`为True。
* 为了节省磁盘空间，缓存镜像在使用后默认将从远程节点中删除（因为镜像在传输过程中需要压缩和解压缩，此处删除的是传输到远程节点的镜像压缩文件），设置`download_keep_remote_cache`可以保留文件。这在开发kubespray时很有用，因为这可以减少下载镜像的时间。因此，远程节点上镜像所需的存储空间将从150MB增加到大约550MB，这是目前所有所需容器镜像的总大小。

容器镜像和二进制文件通过一些变量描述，例如foo_version，foo_download_url，foo_checksum用于描述二进制文件，foo_image_repo, foo_image_tag或foo_digest_checksum（可选）用于描述容器。

容器镜像可以通过它的repo和tag来定义，例如：`andyshinn/dnsmasq:2.72`，或者通过repo和sha256：`andyshinn/dnsmasq@sha256:7c883354f6ea9876d176fe1d30132515478b2859d6fc0cbf9223ffdc09168193`。

注意，必须同时指定SHA256和镜像tag并相互对应。上面给定的示例由以下变量表示：

```yaml
dnsmasq_digest_checksum: 7c883354f6ea9876d176fe1d30132515478b2859d6fc0cbf9223ffdc09168193 
dnsmasq_image_repo: andyshinn/dnsmasq 
dnsmasq_image_tag: '2.72'
```

在download role默认值中可以找到变量的完整列表，还允许为二进制文件和容器镜像指定自定义url和本地存储库。

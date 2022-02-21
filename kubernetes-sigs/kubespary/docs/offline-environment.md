# 离线环境

如果服务器无法访问互联网（例如在有安全限制的本地部署时），需要：

* 一个HTTP反向代理/缓存/镜像，用于提供静态文件（zip和二进制文件）服务
* 一个内部Yum/Deb源
* 一个提供kubespray所有容器镜像的内部仓库，容器镜像列表取决于具体的设置
* [可选]一个用于kubespray所需python包的内部PyPi服务器（仅当操作系统不提供`requirements.txt`列出的所有python包/版本时才需要）
* [可选]一个内部Helm仓库（仅在`helm_enabled=true`时才需要）

## 配置Inventory

一旦所有组件都可以从内网访问，接下来调整inventory中的以下变量以匹配离线环境：

```yaml
# Registry overrides
kube_image_repo: "{{ registry_host }}"
gcr_image_repo: "{{ registry_host }}"
docker_image_repo: "{{ registry_host }}"
quay_image_repo: "{{ registry_host }}"

kubeadm_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubeadm"
kubectl_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubectl"
kubelet_download_url: "{{ files_repo }}/kubernetes/{{ kube_version }}/kubelet"
# etcd is optional if you **DON'T** use etcd_deployment=host
etcd_download_url: "{{ files_repo }}/kubernetes/etcd/etcd-{{ etcd_version }}-linux-amd64.tar.gz"
cni_download_url: "{{ files_repo }}/kubernetes/cni/cni-plugins-linux-{{ image_arch }}-{{ cni_version }}.tgz"
crictl_download_url: "{{ files_repo }}/kubernetes/cri-tools/crictl-{{ crictl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"
# If using Calico
calicoctl_download_url: "{{ files_repo }}/kubernetes/calico/{{ calico_ctl_version }}/calicoctl-linux-{{ image_arch }}"
# If using Calico with kdd
calico_crds_download_url: "{{ files_repo }}/kubernetes/calico/{{ calico_version }}.tar.gz"

# CentOS/Redhat/AlmaLinux/Rocky Linux
## Docker / Containerd
docker_rh_repo_base_url: "{{ yum_repo }}/docker-ce/$releasever/$basearch"
docker_rh_repo_gpgkey: "{{ yum_repo }}/docker-ce/gpg"

# Fedora
## Docker
docker_fedora_repo_base_url: "{{ yum_repo }}/docker-ce/{{ ansible_distribution_major_version }}/{{ ansible_architecture }}"
docker_fedora_repo_gpgkey: "{{ yum_repo }}/docker-ce/gpg"
## Containerd
containerd_fedora_repo_base_url: "{{ yum_repo }}/containerd"
containerd_fedora_repo_gpgkey: "{{ yum_repo }}/docker-ce/gpg"

# Debian
## Docker
docker_debian_repo_base_url: "{{ debian_repo }}/docker-ce"
docker_debian_repo_gpgkey: "{{ debian_repo }}/docker-ce/gpg"
## Containerd
containerd_debian_repo_base_url: "{{ ubuntu_repo }}/containerd"
containerd_debian_repo_gpgkey: "{{ ubuntu_repo }}/containerd/gpg"
containerd_debian_repo_repokey: 'YOURREPOKEY'

# Ubuntu
## Docker
docker_ubuntu_repo_base_url: "{{ ubuntu_repo }}/docker-ce"
docker_ubuntu_repo_gpgkey: "{{ ubuntu_repo }}/docker-ce/gpg"
## Containerd
containerd_ubuntu_repo_base_url: "{{ ubuntu_repo }}/containerd"
containerd_ubuntu_repo_gpgkey: "{{ ubuntu_repo }}/containerd/gpg"
containerd_ubuntu_repo_repokey: 'YOURREPOKEY'
```

对于特定于操作系统只需定义与您的操作系统匹配的设置，如果使用上述设置，则需要在inventory中定义以下变量：

* `registry_host`: 容器镜像仓库，如果`download/role/defaults`中定义的容器镜像使用不相同的仓库，则需要覆盖所有的*_image_repo。如果想让工作更轻松，请使用相同的镜像仓库，这样无需覆盖其他任何内容。
* `files_repo`: 能够提供上述文件的HTTP网络服务器或反向代理。路径并不重要，可以将它们存储在任何地方，只要可以被kubespray访问。建议*_version在路径中使用，这样就无需在每次kubespray升级这些组件之一时修改此设置。
* yum_repo/debian_repo/ubuntu_repo: 操作系统相关的包源，应该指向的内部源。

## 安装Kubespray Python包

### 推荐方式：Kubespray容器镜像

最简单的方法是使用kubespray容器镜像，因为所有需要的包都安装在镜像中，只需将容器镜像复制到私有仓库中即可！

### 手动安装

查看`requirements.txt`文件并检查您的操作系统是否提供了所有开箱即用的软件包（使用操作系统软件包管理器）。对于那些确实的包，需要使用具有Internet访问权限的代理或在内网中设置一个PyPi服务器来托管这些包。

如果使用HTTP(S)代理下载python包：

```bash
sudo pip install --proxy=https://[username:password@]proxyserver:port -r requirements.txt
```

使用内部PyPi服务器：

```bash
# If you host all required packages
pip install -i https://pypiserver/pypi -r requirements.txt

# If you only need the ones missing from the OS package manager
pip install -i https://pypiserver/pypi package_you_miss
```

## 像往常一样运行Kubespray

一旦所有工件都到位并且您的inventory正确设置，您可以使用常规cluster.yaml命令运行kubespray：

```bash
ansible-playbook -i inventory/my_airgap_cluster/hosts.yaml -b cluster.yml
```

如果使用[Kubespray容器镜像](#推荐方式：Kubespray容器镜像)，可以将inventory挂载到容器内：

```bash
docker run --rm -it -v path_to_inventory/my_airgap_cluster:inventory/my_airgap_cluster myprivateregisry.com/kubespray/kubespray:v2.14.0 ansible-playbook -i inventory/my_airgap_cluster/hosts.yaml -b cluster.yml
```

# 通过部署学习Kubernetes之安装cni

## 前言

关于CNI建议先阅读[Kubernetes网络插件](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)和[CNI规范](https://github.com/containernetworking/cni/blob/master/SPEC.md)，对CNI有一定的了解后，再阅读本文会比较容易理解。

本文引用源码为kubespray:v2.18.0版本，因为kubespray支持多种类型操作系统，本文只针对CentOS做解析，其他发行版读者按需自行分析。

## 安装CNI

CNI是容器网络接口，属于规范或协议类，理论上规范或者协议不存在安装一说，我们从代码上看一下安装CNI到底安装了什么？源码链接：<https://github.com/kubernetes-sigs/kubespray/blob/v2.18.0/roles/network_plugin/cni/tasks/main.yml>

```yaml
---
# 确保/opt/cni/bin文件夹存在，因为这是CNI插件二进制文件存放的目录。
# 此处有个问题，为什么CNI二进制文件目录是hardcode在代码中，而不是使用变量？
# 首先，/opt/cni/bin不是CNI规定的；
# 其次，CNI二进制文件目录是kubelet传给容器运行时（比如containerd）;
# 再者，kubelet的CNI二进制文件目录默认值是/opt/cni/bin，可以通过--cni-bin-dir命令行参数修改
# 最后，kubespray没有为CNI二进制文件目录设置变量，直接使用了kubelet的默认值，说明/opt/cni/bin得到的大家的共识
- name: CNI | make sure /opt/cni/bin exists
  file:
    path: /opt/cni/bin
    state: directory
    mode: 0755
    owner: kube
    recurse: true

# 拷贝CNI插件，CNI插件在执行下载阶段的时候就被下载到{{ local_release_dir }}目录（默认为/tmp/releases）。
# 此时只需要将下载的压缩包解压到/opt/cni/bin目录即可。
- name: CNI | Copy cni plugins
  unarchive:
    src: "{{ local_release_dir }}/cni-plugins-linux-{{ image_arch }}-{{ cni_version }}.tgz"
    dest: "/opt/cni/bin"
    mode: 0755
    remote_src: yes
```

## 总结

1. CNI虽然是容器网络接口，但是CNI项目不仅维护接口规范，同时维护了一些[CNI插件](https://github.com/containernetworking/plugins)，这些插件不仅可以作为实现CNI的参考，同时实现了绝大部分插件都必须要实现的部分功能（比如IP地址管理），这样就免去了开发网络插件的重复工作。
2. 安装CNI其实就是安装CNI项目维护的CNI插件。

# 通过部署学习Kubernetes之安装[runc](https://github.com/opencontainers/runc)

## 前言

[runc](https://github.com/opencontainers/runc)是一个CLI工具，[OCI](https://github.com/opencontainers)的一种实现，用于在Linux上生成和运行容器。containerd通过runc运行容器，所以在[安装containerd](../containerd/README.md)之前，需要安装runc。

本文引用源码为kubespray:v2.18.0版本，因为kubespray支持多种类型操作系统，本文只针对CentOS做解析，其他发行版读者按需自行分析。

## 安装[runc](https://github.com/opencontainers/runc)

安装runc就是下载runc的二进制文件到运行目录即可，而且没有任何配置文件，就是这么简单。源码链接：<https://github.com/kubernetes-sigs/kubespray/blob/v2.18.0/roles/container-engine/runc/tasks/main.yml>

```yaml
---
# 与CentOS无关
- name: runc | set is_ostree
  set_fact:
    is_ostree: "{{ ostree.stat.exists }}"

# 通过包管理器删除runc，比如yum remove runc。
# 因为以前的版本是通过包管理器安装runc，当前版本是直接下载二进制文件。
# 此处是避免与其他方式安装的runc冲突，所以先删除通过包管理器安装的runc。
- name: runc | Uninstall runc package managed by package manager
  package:
    name: "{{ runc_package_name }}"
    state: absent
  when:
    - not (is_ostree or (ansible_distribution == "Flatcar Container Linux by Kinvolk") or (ansible_distribution == "Flatcar"))

# 下载runc二进制文件
- name: runc | Download runc binary
  include_tasks: "../../../download/tasks/download_file.yml"
  vars:
    # 下载runc的配置后文有解析
    download: "{{ download_defaults | combine(downloads.runc) }}"

# 将下载的runc二进制文件拷贝到运行目录(比如/usr/bin)，并重命名成runc
- name: Copy runc binary from download dir
  copy:
    src: "{{ downloads.runc.dest }}"
    dest: "{{ runc_bin_dir }}/runc"
    mode: 0755
    remote_src: true

# 因为kubespray默认安装的runc目录为/usr/bin（绝大部分runc的安装路径），但是可以通过配置修改。
# 如果/usr/bin目录下已经安装了runc，而配置的runc安装目录不是/usr/bin则需要删除已经安装的runc
- name: runc | Remove orphaned binary
  file:
    path: /usr/bin/runc
    state: absent
  when: runc_bin_dir != "/usr/bin"
  ignore_errors: true  # noqa ignore-errors
```

接下来再看看runc的下载配置，源码链接：<https://github.com/kubernetes-sigs/kubespray/blob/v2.18.0/roles/download/defaults/main.yml#930>

```yaml
downloads:
  runc:
    # 下载文件
    file: true
    # 只有容器引擎是containerd时才需要下载runc
    enabled: "{{ container_manager == 'containerd' }}"
    version: "{{ runc_version }}"
    # runc下载到指定的目录，默认为/tmp/releases
    dest: "{{ local_release_dir }}/runc"
    # runc二进制文件的校验和
    sha256: "{{ runc_binary_checksum }}"
    # runc下载url，默认为https://github.com/opencontainers/runc/releases/download/{{ runc_version }}/runc.{{ image_arch }}
    url: "{{ runc_download_url }}"
    # 不需要解压
    unarchive: false
    # 用户为root
    owner: "root"
    # 所有用户都可以读和运行，只有root可以写
    mode: "0755"
    # kubernetes所有节点都需要下载runc
    groups:
    - k8s_cluster
```

## 总结

安装runc的流程如下：

1. 删除通过包管理器安装的runc
2. 下载runc

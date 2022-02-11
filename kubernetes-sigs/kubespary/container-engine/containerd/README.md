
# 通过部署学习Kubernetes之安装[containerd](https://github.com/containerd/)

## 前言

如果不知道为什么安装containerd，请先阅读[Kubernetes的容器运行时](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)。

本文引用源码为kubespray:v2.18.0版本，因为kubespray支持多种类型操作系统，本文只针对CentOS做解析，其他发行版读者按需自行分析。

## 安装[containerd](https://github.com/containerd/)

源码链接：<https://github.com/kubernetes-sigs/kubespray/blob/v2.18.0/roles/container-engine/containerd/tasks/main.yml>

```yaml
---
# 检查操作系统发行版是否可以安装containerd
- name: Fail containerd setup if distribution is not supported
  fail:
    msg: "{{ ansible_distribution }} is not supported by containerd."
  when:
    # 如果选择的操作系统发行版不在如下列表，需要自行添加到，否则无法安装
    - ansible_distribution not in ["CentOS", "OracleLinux", "RedHat", "Ubuntu", "Debian", "Fedora", "AlmaLinux", "Rocky", "Amazon", "Flatcar", "Flatcar Container Linux by Kinvolk", "Suse", "openSUSE Leap", "openSUSE Tumbleweed"]

# 通过包管理器删除已安装的contrainerd，比如yum remove containerd。
# kubespray以前的版本是通过包管理器安装containerd，当前版本是直接下载containerd的二进制包。
# 安装新的containerd需要确保节点上没有其他方式安装的containerd
- name: containerd | Remove any package manager controlled containerd package
  package:
    name: "{{ containerd_package }}"
    state: absent
  when:
    - not (is_ostree or (ansible_distribution == "Flatcar Container Linux by Kinvolk") or (ansible_distribution == "Flatcar"))

# 删除containerd的yum源配置，当前版本已经不在使用包管理器安装containerd
- name: containerd | Remove containerd repository
  file:
    path: "{{ yum_repo_dir }}/containerd.repo"
    state: absent
  when:
    - ansible_os_family in ['RedHat']

# 与CentOS无关
- name: containerd | Remove containerd repository
  apt_repository:
    repo: "{{ item }}"
    state: absent
  with_items: "{{ containerd_repo_info.repos }}"
  when: ansible_pkg_mgr == 'apt'

# 下载containerd二进制文件
- name: containerd | Download containerd
  include_tasks: "../../../download/tasks/download_file.yml"
  vars:
    # 下载配置后文有解析
    download: "{{ download_defaults | combine(downloads.containerd) }}"

# 解压缩containerd
- name: containerd | Unpack containerd archive
  unarchive:
    src: "{{ downloads.containerd.dest }}"
    dest: "{{ containerd_bin_dir }}"
    mode: 0755
    remote_src: yes
    extra_opts:
      - --strip-components=1
  notify: restart containerd

# containerd的运行目录可以配置，默认的运行目录是/usr/bin，如果配置的运行目录不是/usr/bin
# 需要将安装在/usr/bin目录下的containerd程序删除
- name: containerd | Remove orphaned binary
  file:
    path: "/usr/bin/{{ item }}"
    state: absent
  when: containerd_bin_dir != "/usr/bin"
  ignore_errors: true  # noqa ignore-errors
  with_items:
    - containerd
    - containerd-shim
    - containerd-shim-runc-v1
    - containerd-shim-runc-v2
    - ctr

# 生成containerd的服务
- name: containerd | Generate systemd service for containerd
  template:
    src: containerd.service.j2
    dest: /etc/systemd/system/containerd.service
    mode: 0644
  notify: restart containerd

# 确保containerd的目录存在
- name: containerd | Ensure containerd directories exist
  file:
    dest: "{{ item }}"
    state: directory
    mode: 0755
    owner: root
    group: root
  with_items:
    # containerd服务路径，默认为/etc/systemd/system/containerd.service.d
    - "{{ containerd_systemd_dir }}"
    # containerd配置目录，默认为/etc/containerd
    - "{{ containerd_cfg_dir }}"
    # containerd存储镜像的目录，默认为/var/lib/containerd
    - "{{ containerd_storage_dir }}"
    # containerd的容器目录，默认为/run/containerd
    - "{{ containerd_state_dir }}"

# http/https代理相关的本文不关注，默认http/https代理关闭即可
- name: containerd | Write containerd proxy drop-in
  template:
    src: http-proxy.conf.j2
    dest: "{{ containerd_systemd_dir }}/http-proxy.conf"
    mode: 0644
  notify: restart containerd
  when: http_proxy is defined or https_proxy is defined

# 复制containerd配置文件，一般生成配置的方法是containerd config default > /etc/containerd/config.toml
# 然后再默认参数的基础上修改部分自定义参数
- name: containerd | Copy containerd config file
  template:
    src: config.toml.j2
    dest: "{{ containerd_cfg_dir }}/config.toml"
    owner: "root"
    mode: 0640
  # 为了保证新的配置生效，需要重启containerd服务
  notify: restart containerd

# you can sometimes end up in a state where everything is installed
# but containerd was not started / enabled
# 
- name: containerd | Flush handlers
  meta: flush_handlers

# 确保containerd服务启动
- name: containerd | Ensure containerd is started and enabled
  service:
    name: containerd
    enabled: yes
    state: started
```

## 总结

安装containerd的流程如下：

1. 校验操作系统的发行版是否在指定的范围内；
2. 删除通过包管理器安装的containerd
3. 下载containerd;
4. 生成containerd服务；
5. 确保containerd服务运行；

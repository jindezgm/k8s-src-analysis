# 通过部署学习Kubernetes之关闭交换空间

## 前言

Kubernetes从1.8版本之后必须关闭系统[交换空间](https://www.linux.com/news/all-about-linux-swap-space/)，否则将不能工作（kubelet）。本文将通过kubespray源码分析如何关闭系统交换空间，更重要的是分析为什么要关闭交换空间。

本文引用源码为kubespray:v2.18.0版本，操作系统类型只针对CentOS做解析，其他发行版读者按需自行分析。

## 关闭系统[交换空间](https://www.linux.com/news/all-about-linux-swap-space/)

关闭系统[交换空间](https://www.linux.com/news/all-about-linux-swap-space/)是kubespray的一个[role](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)， 源码链接：<https://github.com/kubernetes-sigs/kubespray/blob/v2.18.0/roles/kubernetes/preinstall/tasks/0010-swapoff.yml>

```yaml
---
# 卸载(unmount)swap类型设备，同时删除/etc/fstab中配置的交换设备/文件
# 这个操作是非常必要的，虽然后面会通过swapoff -a禁止交换，但没有这个操作，系统重启后交换空间还是会开启的
- name: Remove swapfile from /etc/fstab
  mount:
    name: "{{ item }}"
    fstype: swap
    state: absent
  with_items:
    - swap
    - none

# swapon -s可以列举正在使用的交换设备/文件的信息，如果开启了系统交换空间，swapon变量就会被赋值
# swapon变量就可用来判断是否开启了系统交换空间，如果开启了，需要关闭它
- name: check swap
  command: /sbin/swapon -s
  register: swapon
  changed_when: no

# swapoff -a将/etc/fstab文件中所有设置为swap的设备禁止交换，关于swapoff后文有详细说明
- name: Disable swap
  command: /sbin/swapoff -a
  when:
    # swapon是swapon -s的输出值，这个条件说明swapon -s有输出，也就是说明当前有使用的交换设备/文件
    - swapon.stdout
    # 开启swap时kubelet默认是报错的，如果非要强制开启swap，则需要配置kubelet，如何配置笔者会在部署kubelet的文章中说明。
    - kubelet_fail_swap_on | default(True)
  ignore_errors: "{{ ansible_check_mode }}"  # noqa ignore-errors

# Fedora不在本文解析范围
- name: Disable swapOnZram for Fedora
  command: touch /etc/systemd/zram-generator.conf
  when:
    - swapon.stdout
    - ansible_distribution in ['Fedora']
    - kubelet_fail_swap_on | default(True)
```

[swapoff](https://linux.die.net/man/8/swapoff)在指定的设备和文件上禁用swap，当使用-a参数时，在所有已知的交换设备和文件（如/proc/swaps或[/etc/fstab](https://www.redhat.com/sysadmin/etc-fstab)中的配置的）上禁用交换。

## 总结

关闭系统[交换空间](https://www.linux.com/news/all-about-linux-swap-space/)其实很简单：1）卸载交换设备并删除[/etc/fstab](https://www.redhat.com/sysadmin/etc-fstab)中的相关配置；2）通过swapoff -a在已知的交换设备/文件禁止swap。

那么问题来了，Kubernetes为什么要禁止交换空间呢？关于这个问题，很早就有人[开贴讨论](https://serverfault.com/questions/881517/why-disable-swap-on-kubernetes)，github上也有相关的[issue](https://github.com/kubernetes/kubernetes/issues/53533)。笔者结合大家的讨论以及个人的理解，总结如下：

1. swap是让操作系统为应用程序提供了超出物理内存大小（比如10倍于物理内存）的虚拟内存，这在物理内存非常稀有的以前非常有用，当然现在依然非常有用。但是Kubernetes的思想是容器的CPU/内存应该固定在资源限制(容器资源的limit)范围内，所以根本不需要交换，交换反而会降低性能。
2. 有不少人认为关闭swap是Kubernetes团队的懒惰行为，一刀切的行为并不可取，非常典型的案例就是：应用虽然需要40GB的内存，但是大部分数据都很少访问，只有很少的数据是频繁访问的，开启swap可以让应用在16GB内存的主机中运行；关闭swap就必须在64GB的主机中运行，这会增加成本。
3. 这就是以前运行比较正常的服务，到了Kubernetes上后会OOM的原因，因为没有swap提供更大的虚拟内存。
4. 笔者更倾向于认为支持swap当前还不是Kubernetes的重点，但允许用户通过配置开启swap（参看kubelet的配置），当然后果需要自己承担，毕竟官方不建议开启。

因为对相关内容学习时间较短，以上总结可能不准确也不全面，随着笔者更深入的理解，会逐渐完善。

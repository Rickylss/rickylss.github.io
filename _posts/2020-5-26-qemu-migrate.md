---
layout: post
title:  "libvirt&QEMU 热迁移"
subtitle: ""
date:   2020-5-26 19:13:45 +0800
tags:
  - qemu
  - mem
categories: [QEMU]
comment: true
---

迁移虚拟机对云平台来说是一个非常基础的功能，但是虚拟机的迁移中包含了很多细节和限制条件，这些问题都由云平台为我们解决了。想要深入了解虚拟机迁移的过程，那就需要掌握这些细节。
<!-- more -->
# 1、 简介

QEMU 支持`save/load`运行中的 VM 的状态，`save`即是保存 VM 上所有设备的状态，`load`则反过来加载 VM 上所有设备的状态。

基于`savevm/loadvm`功能，催生除了一个全新的功能——`offline migration`。这意味着，QEMU 可以启动一个 VM 然后将它迁移到另一台物理机上。

紧接着`live migration`功能被提出，这个功能十分重要，因为有些运行中的 VM 携带了大量的设备状态（尤其是 RAM），要将所有状态转移到另一台物理机要花费大量的时间。`live migration`允许 VM 在迁移时任然能够允许，保证服务不中断。只有到最后阶段才会停止，通常情况下，VM 在`live migration`过程中没有响应的时间只有几百毫秒。

qemu-monitor 中提供了 Migration 命令。使用者可以通过该命令将 VM 从源主机（以下简称 srcHost）迁移到目标主机（以下简称 dstHost），迁移成功后，VM 将继续在目标主机上运行。

>KVM 支持在 AMD 主机和 Intel 主机之间互相迁移，通常来说，64 位 VM 只能够迁移到 64 位的主机上，但是 32 位的主机可以随意迁移。
>
>有一些旧的 Intel 处理器不支持 NX(或者 XD)，这可能导致从支持 NX 主机上迁移虚拟机到不支持 NX 主机上出问题。解决方法是在启动 VM 时使用参数`-cpu qemu64,-nx`来关闭 NX。

# 2、要求

要迁移一台 VM，需要 srcHost 和 dstHost 满足一些要求：

- VM 镜像要能够同时被 srcHost 和 dstHost 访问（镜像可放在如 nfs 等共享存储中）；

> 其实这一条是要求最终 VM 镜像相同，而并不强制要求 VM 镜像为同一个镜像。QEMU 中支持 migrate [-b] [-i]。其中：
>
> [-b] for migration without shared storage with full copy of disk
>
> ​		用于在没有具有磁盘完整副本的共享存储时迁移。
>
> [-i] for migration without shared storage with incremental copy of disk (base image shared between src and destination)
>
> ​		用于在没有具备磁盘增量副本的共享存储时迁移 (在 src 和目标之间共享基本映像)

>补充：新版本的 QEMU 编译时不支持老的 blk、inc 方法，提供了 driver_mirror+NBD 的方式（同时也是 libvirt 使用的方式）。
>
>QEMU compiled without old-style (blk/-b, inc/-i) block migration Use drive_mirror+NBD instead.

- 建议放置 VM 镜像的文件夹在两个主机上的路径相同，比如都在`/tmp/nfs/share/`目录下（用于迁移写时拷贝的镜像——使用`qemu-image create -b...`命令在基础镜像上创建的镜像）；
- 保证 srcHost 和 dstHost 在同一个子网中（为了保证 VM 的 tap 网络可用）；
- 不要使用`-snapshot`命令行选项；
- VM 必须在 dstHost 上以和在 srcHost 上同样的方式启动。

> 在这里需要源主机和目标主机以同样（QEMU 命令行选项相同）的方式启动同一个 VM 镜像，这样做的目的是为了保证在目标主机上启动的 VM（以下简称 dstVM）和源主机上的 VM（以下简称 srcVM）直接所拥有的设备相同，否则无法保证 VM 的状态一致。
>
> 对`-snapshot`和写时拷贝镜像的限制同理，也是为了防止内存更改不能同步的问题。

值得注意的是，事实上对 srcVM 和 dstVM 的启动参数并没有那么严格，参考 virsh 源码中`migration protocols 3`的做法，如 CPU 和内存数等部分参数是可以更改的。

>virsh 中提出了 3 中`migration protocols`（src/qemu/MIGRATION.txt），其中对 QEMU 提供了`migration protocols 2\3`的支持，未能支持`migration protocols 1`。在后文的 libvirt 代码分析中将详细说明。

# 3、QEMU 热迁移

我们可以通过 qemu-monitor 来迁移一台 VM，直接使用 hmp 或者使用 qmp 协议来迁移一台 VM。

## 3.1、迁移方式

迁移数据流是一个能够通过任何方式传输的字节流。

- tcp migration：通过 tcp sockets 迁移；
- unix migration：通过 unix sockets 迁移；
- exec migration：通过进程的标准输入输出（stdin/stdout）迁移；
- fd migration：通过文件描述符迁移，QEMU 不在乎这个文件描述符是怎么打开的；

另外，通过 RDMA 的方式迁移也是支持的，这时由硬件来管理内存页的传输，同时也减轻了 CPU 的负载。虽然 RDMA 迁移方式的内部实现机制略有不同，但是它在除 RAM 迁移代码之外的地方是透明的。

以上所有迁移协议都使用相同的基础设施来 save/restore 设备的状态。这个写基础设施功能与 savevm/loadvm 共享。

## 3.2、调试

通过`scripts/analyze_migration.py`脚本我们可以分析迁移数据流。

```bash
$ qemu-system-x86_64 ....
(qemu) migrate "exec:cat > mig"
$ ./scripts/analyze_migration.py -f mig
{
  "ram(3)": {
    "section sizes":{
      "pc.ram": "0x000000000000",
......
    }
  }
}
```

通过`analyze_migration.py -h`命令查看更多选项。

## 3.3、迁移实验

1. 实验条件：两台物理机（或者两台虚拟机，装作物理机）

   略；

2. 安装 virsh、QEMU 和 nfs

   略；

3. 配置 nfs，准备一个 qcow2 镜像，

   ```bash
   $ vim /etc/exports
   
   /path_to_share/ *(rw,no_root_squash,async)
   
   # 注意如果不设置防火墙，可能无法远程 mount 共享文件夹
   $ firewall-cmd --permanent --add-service=nfs
   $ firewall-cmd --permanent --add-service=rpc-bind
   $ firewall-cmd --permanent --add-service=mountd
   $ firewall-cmd --reload
   $ systemctl restart rpcbind
   $ systemctl restart nfs
   $ systemctl enable rpcbind && systemctl enable nfs
   $ exportfs -r
   $ showmount -e
   ```

4. 做一个 share.img，放入写内存的脚本

   ```bash
   $ cat mem_test.c
   
   include <stdlib.h>
   include <stdio.h>
   int main()
   {
       char *buf = (char *) calloc(4096, 4096);
       while (1) {
           int i;
           for (i = 0; i < 4096 * 4; i++) {
               buf[i * 4096 / 4]++;
           }
           printf(".");
       }
   }
   $ gcc mem_test.c -o mem_test
   $ dd if=/dev/zero of=/path/to/share.img bs=1024k count=1000
   $ mkfs.ext4 /path/to/share.img
   $ mount /path/to/share.img /mnt
   $ mv mem_test /mnt
   $ unmout /mnt
   $ qemu-img convert -O qcow2 /path/to/share.img /path/to/share.qcow2
   ```

5. 分别启动虚拟机

   ```bash
   # srcHost
   $ /usr/libexec/qemu-kvm -machine pc-i440fx-rhel7.6.0,accel=kvm -hda /mnt/902f9d2d469345b7a7364bd774b60802.qcow2 -hdb /mnt/share.qcow2 -chardev pty,id=charserial0 -device isa-serial,chardev=charserial0,id=serial0 --monitor stdio
   
   # dstHost
   $ /usr/libexec/qemu-kvm -machine pc-i440fx-rhel7.6.0,accel=kvm -hda /mnt/902f9d2d469345b7a60802.qcow2 -hdb /mnt/share.qcow2 -chardev pty,id=charserial0 -device isa-serial,chardev=charserial0,id=serial0 --dio -incoming tcp:0:4444
   ```

6. 连接到 srcVM 中在 srcVM 中启动脚本

   ```bash
   $ ./mem_test
   ```

7. 开始 migration

   ```bash
   # srcHost
   # 查看 migration 状态
   (qemu) info migrate
   globals:
   store-global-state: on
   only-migratable: off
   send-configuration: on
   send-section-footer: on
   decompress-error-check: on
   # 查看 capabilities
   (qemu) info migrate_capabilities
   xbzrle: off
   rdma-pin-all: off
   auto-converge: off
   zero-blocks: off
   compress: off
   events: off
   postcopy-ram: off
   x-colo: off
   release-ram: off
   return-path: off
   pause-before-switchover: off
   x-multifd: off
   dirty-bitmaps: off
   late-block-activate: off
   # 开始迁移
   (qemu) migrate -d tcp:dstHost_ip:4444
   # 查看迁移进度
   (qemu) info migrate
   
   globals:
   store-global-state: on
   only-migratable: off
   send-configuration: on
   send-section-footer: on
   decompress-error-check: on
   capabilities: xbzrle: off rdma-pin-all: off auto-converge: off zero-blocks: off compress: off events: off postcopy-ram: off x-colo: off release-ram: off return-path: off pause-before-switchover: off x-multifd: off dirty-bitmaps: off late-block-activate: off
   Migration status: completed
   total time: 4955 milliseconds
   downtime: 139 milliseconds
   setup: 7 milliseconds
   transferred ram: 167038 kbytes
   throughput: 276.70 mbps
   remaining ram: 0 kbytes
   total ram: 148296 kbytes
   duplicate: 6845 pages
   skipped: 0 pages
   normal: 41663 pages
   normal bytes: 166652 kbytes
   dirty sync count: 5
   page size: 4 kbytes
   
   # 调试迁移数据流
   (qemu) migrate "exec:cat > mig"
   ```

   完成迁移后，会发现 srcVM 已关闭，打开 dstVM，发现其开始运行`./mem_test`程序。

## 3.4、问题

- TSC offset on the new host must be set in such a way that the guest sees a monotonically increasing TSC, otherwise the guest may hang indefinitely after migration.
- usbdevice tablet complains after migration.
- handle migration completion better (especially when network problems occur).
- More informative status.
- Migration does not work while CPU real-mode/protected mode are still changing.
- Migration does not work when running [Nested Guests](https://www.linux-kvm.org/page/Nested_Guests): if you are running a KVM guest that is itself running a KVM guest, migration of the unnested guest (including savevm/loadvm as described below) will result in undefined behavior.

## 3.5、拓展

通过 exec 标准输入输出迁移

```bash
# srcHost
# 通过标准输入输出必须先把虚拟机暂停
(qemu) stop
(qemu) migrate_set_speed 4095m
(qemu) migrate "exec:gzip -c > STATEFILE.gz"
```

```bash
# dstHost
$ gzip -c -d STATEFILE.gz | <qemu-command-line> -incoming "exec: cat"
$ <qemu-command-line> -incoming "exec: gzip -c -d STATEFILE.gz"
```

或者通过加密的方式：

```bash
# srcHost
# 通过标准输入输出必须先把虚拟机暂停
(qemu) stop                                                                          
(qemu) migrate_set_speed 4095m
(qemu) migrate "exec:gpg -q -e -r KEY -o STATFILE.gpg"
```

```bash
# dstHost
$ gpg -q -d -r KEY STATEFILE.gpg | <qemu-command-line> -incoming "exec:cat"
```

调试迁移数据流的方法就是将迁移数据流重定向到一个文件里

```bash
# 调试迁移数据流
(qemu) migrate "exec:cat > mig"
```

# 4、libvirt 迁移

libvirt 迁移可使用`virsh migrate`命令，略。

# 5、precopy 和 postcopy

迁移有两种技术实现。可参考：

https://en.wikipedia.org/wiki/Live_migration

https://libvirt.org/html/libvirt-libvirt-domain.html#virDomainMigrateStartPostCopy

# 6、迁移算法

1. 在 detHost 开启 detVM，并准备接受迁移数据流，开启脏页记录；
2. 迁移内存，
   - srcVM 继续运行
   - 设置好迁移数据流带宽
   - 首先将整个内存迁移
   - 再迁移脏页数据
3. 停止 srcVM；
4. 迁移虚拟机所有设备状态，所有剩下的设备状态和内存脏页都在此时传输，不限制迁移数据流带宽；
5. 启动 detVM；
   - 广播”I'm over here“数据包，宣布虚拟机新网卡的位置；

## 6.1、QEMU 源码分析

待补充。

### 6.1.1、公共基础设置

携带迁移数据流的 files、sockets 或者 fd，在 QEMU 中都使用`QEMUFile`（see `migration/qemu-file.h`）类型抽象出来。它通常和`QIOChannel`（see `io/`）子类型连接起来。

## 6.2、libvirt 源码分析

待补充。

# reference

KVM: https://www.linux-kvm.org/page/Migration

QEMU(source code): [/docs/devel/migration.rst](https://github.com/qemu/qemu/blob/master/docs/devel/migration.rst)

qemu 热迁移简介：[https://terenceli.github.io/%E6%8A%80%E6%9C%AF/2018/03/01/qemu-live-migration](https://terenceli.github.io/技术/2018/03/01/qemu-live-migration)
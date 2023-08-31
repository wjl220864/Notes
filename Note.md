# 交叉编译DPDK

## 0. Get the dpdk source code

[http://core.dpdk.org/download/](http://core.dpdk.org/download/)

## 1. Getting the prerequisite library

NUMA is required by most modern machines, not needed for non-NUMA architectures.

Note

For compiling the NUMA lib, run libtool –version to ensure the libtool version >= 2.2, otherwise the compilation will fail with errors.

```
git clone https://github.com/numactl/numactl.git
cd numactl
git checkout v2.0.11 -b v2.0.11
./autogen.sh
autoconf -i
./configure --host=aarch64-linux-gnu CC=aarch64-linux-gnu-gcc --prefix=<numa install dir>
make install
```

The numa header files and lib file is generated in the include and lib folder respectively under <numa install dir>.



## 2. Augment the cross toolchain with NUMA support

Note

This way is optional, an alternative is to use extra CFLAGS and LDFLAGS, depicted in [Configure and cross compile DPDK Build](https://doc.dpdk.org/guides-19.08/linux_gsg/cross_build_dpdk_for_arm64.html#configure-and-cross-compile-dpdk-build) below.

Copy the NUMA header files and lib to the cross compiler’s directories:

```
cp <numa_install_dir>/include/numa*.h <cross_install_dir>/gcc-arm-8.2-2019.01-x86_64-aarch64-linux-gnu/bin/../aarch64-linux-gnu/libc/usr/include/
cp <numa_install_dir>/lib/libnuma.a <cross_install_dir>/gcc-arm-8.2-2019.01-x86_64-aarch64-linux-gnu/lib/gcc/aarch64-linux-gnu/7.5.0/
cp <numa_install_dir>/lib/libnuma.so <cross_install_dir>/gcc-arm-8.2-2019.01-x86_64-aarch64-linux-gnu/lib/gcc/aarch64-linux-gnu/7.5.0/
```



## 3. Configure and cross compile DPDK Build

- 设置环境变量

  ```shell
  export PATH=$PATH:<交叉编译工具链绝对路径> //例：(/home/.../bin)
  export CROSS=aarch64-linux-gnu-
  export RTE_KERNELDIR=<内核路径>
  export CROSS_COMPILE=aarch64-linux-gnu-
  export EXTRA_CFLAGS="-isystem <numa install dir>/include" 
  export EXTRA_LDFLAGS="-L<numa install dir>/lib -lnuma"
  export RTE_SDK=`pwd`//dpdk顶层目录
  export RTE_TARGET=build
  ```
  
- 配置

```shell
make config T=arm64-armv8a-linuxapp-gcc
```

- 编译驱动

```shell
make -j 
```

- 编译示例程序（以examples/exception_path为例）

```shell
cd examples/exception_path
make -j
```

# linux查看GPIO编号

**内核配置**

内核需要打开debugfs支持

```ini
Symbol: DEBUG_FS [=y]
   Prompt: Debug Filesystem
     Defined at lib/Kconfig.debug:77
     Depends on: SYSFS     
     Location:
       -> Kernel configuration
         -> Kernel hacking      
```

**终端操作**

导出gpio端口

```bash
cd /sys/class/gpio
for ((i=480;i<=511;i++)) do echo $i > export; done
```

挂载debugfs

```bash
mount -t debugfs none /sys/kernel/debug
```

打印GPIO DEBUG信息

```bash
cat /sys/kernel/debug/gpio
```

查看gpio各项参数

```bash
for ((i=480;i<=511;i++)) do echo $i : `cat gpio$i/value`; done
for ((i=480;i<=511;i++)) do echo $i : `cat gpio$i/direction`; done
for ((i=480;i<=511;i++)) do echo $i : `cat gpio$i/edge`; done
```

设置gpio中断触发方式

```bash
for ((i=480;i<=511;i++)) do echo gpio$i; echo both > gpio$i/edge; done
```

读取GPIO中断寄存器

```bash
devmem 0x28004018 //GPIOO A组中断寄存器地址
devmem 0x28005018 //GPIO1 A组中断寄存器地址
```



# Linux的网络配置文件

## 1. /etc/network/interfaces 文件语法

```shell
#JARI-WORKS Network Interface File

auto lo
iface lo inet loopback

#不设置IP
auto eth0
iface eth0 inet manual

#设置静态IP
auto eth1
iface eth1 inet static
        address 192.168.1.2
        netmask 255.255.255.0
        gateway 192.168.1.1

#设置自动获取IP
auto eth2
iface eth2 inet dhcp

#配置冗余网口
auto eth2
iface eth2 inet manual
    pre-up ifconfig eth2 up
    bond-master bond0

auto eth3
iface eth3 inet manual
    pre-up ifconfig eth3 up
    bond-master bond0

auto bond0
iface bond0 inet static
    pre-up modprobe bonding mode=1 miimon=100 primary=eth2
    post-up ifenslave bond0 eth2 eth3
    address 192.168.5.204
    netmask 255.255.255.0
    gateway 192.168.5.1
        
```

其中，每个配置块都是由以下几个部分组成：

- auto: 表示该网卡开机时自动启用
- iface: 声明该网卡的名称，可以是lo、eth0、eth1等
- inet: 网络协议族，可以是inet或inet6表示IPv4或IPv6
- method: 配置方法，可以是manual、dhcp、static、loopback等
- address: 指定该网卡的IP地址
- netmask: 指定该网卡的掩码
- gateway: 指定该网卡的网关
- dns-nameservers: 指定DNS服务器地址

## 2. /etc/resolv.conf 文件语法

```shell
nameserver 192.168.1.1
nameserver 8.8.8.8
search example.com
```

其中，每个配置项的含义如下：

- nameserver: 指定DNS服务器地址
- search: 指定默认域名

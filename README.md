# 虚拟机Ubuntu版本
```bash
$ uname -v
#167~18.04.1-Ubuntu SMP Wed May 24 00:51:42 UTC 2023

# 20.04版本编译用户空间应用一直有问题，没编译成功
$ uname -v
#101~20.04.1-Ubuntu SMP Thu Nov 16 14:22:28 UTC 2023
```

# checkout v5.15-dovetail2-rebase
```bash
git clone https://source.denx.de/Xenomai/linux-dovetail.git
cd linux-dovetail
git tag -l
git checkout -b my_branch_v5.15 v5.15-dovetail2-rebase
```

# checkout Linux 5.15.0
```bash
git checkout -b my_branch_v5.15 v5.15-dovetail2-rebase
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.tar.gz
```

# 生成dovetail patch
```bash
sudo diff -uprN /mnt/workspace/Xilinx-soft-2022.2/linux-5.15/ linux-dovetail/ > patch-5.15.0-dovetail.patch
```

# 生成cobalt-core补丁
```bash
./prepare-kernel.sh --linux='/mnt/workspace/Xilinx-soft-2022.2/linux-5.15' --arch=arm64 --outpatch='/mnt/workspace/GitHub/cobalt-core-3.2.x-linux5.15.0.patch'
```

# 打补丁
```bash
cd linux-xlnx-xilinx-v2022.2
patch -p1 < '/mnt/workspace/GitHub/patch-5.15.0-dovetail.patch'
patch -p1 < '/mnt/workspace/GitHub/cobalt-core-3.2.x-linux5.15.0.patch'
```

# 配置内核
```bash
petalinux-config -c kernel
```

```
Go to 
General setup / Timers subsystem / 
and make sure the "High Resolution Timer Support" is selected

Go back to the main menu and to 
Kernel Features / Preemption Model 
Select "Fully Preemptive Kernel (RT)" to activate the linux-rt features
Go back to Kernel Features, select Timer frequency and set it to 1000 Hz
Go back to the main menu and select 

CPU power Manapreegement 
Disable the CPU frequency scaling
```

# 进入menuconfig配置界面会看到xenomai选项有如下警告信息：
```
 *** WARNING! Page migration (CONFIG_MIGRATION) may increase *** 
 *** latency. ***                                                         
 *** WARNING! At least one of APM, CPU frequency scaling, ACPI 'processor' ***
 *** or CPU idle features is enabled. Any of these options may ***     
 *** cause troubles with Xenomai. You should disable them. ***      
```

# 消除该警告信息需做如下配置：
```
CPU Power Management --->
	CPU Frequency scaling  --->
		[] CPU Frequency scaling
	CPU Idle  --->
		[] CPU idle PM support

Memory Management options  --->
	[ ] Contiguous Memory Allocator
	[ ] Transparent Hugepage Support
	[ ] Allow for memory compaction
	[ ] Page migration 
```


# 用户空间应用库编译
```bash
sudo apt-get upgrade
sudo apt-get update
sudo apt-get install gcc-aarch64-linux-gnu
aarch64-linux-gnu-gcc -v

./scripts/bootstrap

./configure CFLAGS="-mtune=cortex-a53" LDFLAGS="-mtune=cortex-a53" --build=i686-pc-linux-gnu --host=aarch64-linux-gnu --with-core=mercury --enable-smp --enable-pshared CC=aarch64-linux-gnu-gcc LD=aarch64-linux-gnu-ld

./configure CFLAGS="-mtune=cortex-a53" LDFLAGS="-mtune=cortex-a53" --build=i686-pc-linux-gnu --host=aarch64-linux-gnu --with-core=cobalt --enable-smp CC=aarch64-linux-gnu-gcc LD=aarch64-linux-gnu-ld

mkdir staging
sudo make DESTDIR=`pwd`/staging install

# 部署到目标板
scp -r staging/usr/xenomai/ root@192.168.1.136:~/
```

# 目标板操作
```bash
mv xenomai /usr

vi /etc/profile

# Export xenomai environment variables
export PATH=/usr/xenomai/bin:$PATH
export PATH=/usr/xenomai/lib:$PATH
export PATH=/usr/xenomai/sbin:$PATH

source /etc/profile

/usr/xenomai/bin# ./xeno-config
/usr/xenomai/bin# ./latency


# 观察到last best项为负值，说明xenomai需要校准
sudo -s
echo 0 > /proc/xenomai/latency
./latency

# 运行一段时间后得到新的lat best值，将这个值*1000
echo 4413 > /proc/xenomai/latency
./latency
```

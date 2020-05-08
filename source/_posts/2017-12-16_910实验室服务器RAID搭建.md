title: 910实验室服务器RAID搭建
date: 2017-12-16 14:03:43
tags: [实验室服务器搭建]
categories: [log]
---
# 实验室服务器RAID卷创建记录
## 创建SSD RAID0
待创建的SSD 为sde与sdf
使用mdadm工具创建卷

<!-- more -->

### 查看待加入的磁盘挂载情况

```
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT
```

### 创建数组

```
sudo mdadm --create --verbose /dev/md0 --level=0 --raid-devices=2 /dev/sde /dev/sdf
```
创建后将清空原有的磁盘分区表，根据提示输入y，即可完成。

### 查看创建结果

```
cat /proc/mdstat
```
根据输出的情况查看RAID版本，以及加入的磁盘是否正确。

### 创建并安装文件系统

1. 
```
sudo mkfs.ext4 -F /dev/md0
```
根据创建的RAID卷号，修改md后的序号，即可在卷上创建ext4文件系统

2. 
```
sudo mkdir -p /mnt/SSD
```
创建挂载点

3. 
```
sudo mount /dev/md0 /mnt/SSD
```
挂载创建好的卷

### 配置自启动

1. 
```
#### sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
```
保存生成的磁盘阵列
### 此命令执行一次就好，否则会在/etc/mdadm/mdadm.conf出现重复项

2. 
```
sudo update-initramfs -u
```
更新initramfs或初始RAM文件系统，以便阵列在早期引导过程中可用

3. 
```
echo '/dev/md0 /mnt/SSD ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
```
添加开机挂载选项

针对SSD的RAID0创建完毕

## 创建HDD的RAID1
待创建的HDD为sdg和sdh
使用mdam创建冗余阵列

### 创建RAID阵列

```
sudo mdadm --create --verbose /dev/md1 --level=1 --raid-devices=2 /dev/sdg /dev/sdh
```
创建的结果添加到 /dev/md1

**创建结束后，一定要通过命令
```
cat /proc/mdstat
```
查看当前的进展，当同步进度为100%时，才可进行后续操作！！！！**

### 创建文件系统并挂载
1. 
```
sudo mkfs.ext4 -F /dev/md1
```
在RAID上创建文件系统

2. 
```
sudo mkdir -p /mnt/HDD
```
创建挂载点，命名为HDD

3. 
```
sudo mount /dev/m1 /mnt/HDD
```
挂载到挂载点

4. 
```
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
```
保存RAID布局

5. 
```
sudo update-initramfs -u
```
更新initramfs或初始RAM文件系统，以便阵列在早期引导过程中可用

6. 
```
echo '/dev/md1 /mnt/HDD ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
```
更新fstab，开机自动挂载


# 所有过程执行完毕
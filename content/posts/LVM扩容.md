---
date: '2026-04-29T20:04:23+08:00'
title: 'LVM扩容'
---

 





## 🧩 什么是 LVM？

LVM 是一种灵活的磁盘管理方式，可以让你：

- 动态调整分区大小（无需重新分区）
- 将多个物理磁盘/分区组合成一个逻辑卷组（VG）
- 在逻辑卷组中创建多个逻辑卷（LV）

------

## ✅ 假设你目前的情况是：

- `/home` 目录当前是挂载在一个普通分区上（比如 `/dev/sda2`）
- 你希望将 `/home` 迁移到 LVM 管理下，并进行扩容

------

## 🔧 步骤：使用 LVM 扩容 `/home`

### 🧰 需要的前提条件：

1. 你有**未使用的磁盘或分区**（例如 `/dev/sdb` 或 `/dev/sdb1`）
2. 安装了 `lvm2` 工具（一般 Ubuntu 默认已安装）
3. apt install lv2 -y 

------

## ✅ 步骤一：准备物理卷（PV）

假设你有一个新的磁盘 `/dev/sdb`，我们可以将其初始化为物理卷：

```
sudo pvcreate /dev/sdb
```

> 如果你已经有分区（如 `/dev/sdb1`），可以用：
>
> ```
> sudo pvcreate /dev/sdb1
> ```

------

## ✅ 步骤二：创建卷组（VG）

创建一个卷组，例如 `myvg`，并将物理卷加入其中：

```
sudo vgcreate myvg /dev/sdb
```

> 如果你已经有一个卷组，可以跳过这一步，直接使用现有的 VG。
>
> sdb      8:16   0   3.7T  0 disk
> └─sdb1   8:17   0   3.7T  0 part
> sdc      8:32   0   3.7T  0 disk
> └─sdc1   8:33   0   3.7T  0 part
>
> 这个是203上，lsblk的磁盘输出，显然每个磁盘下都只有一个分区，那么在创建pv的时候使用哪个呢?
>
> 答：无脑使用sdb1和sdc1，除非你非常自信的使用整块磁盘

------

## ✅ 步骤三：创建逻辑卷（LV）

创建一个逻辑卷，用于存放 `/home` 数据。假设你要创建一个 50GB 的逻辑卷：

```
lvcreate -L 3.5T -n home_lv vg_sdb1_sdc1_per3point7T

sudo lvcreate -L 50G -n home_lv myvg
```

> `-L` 表示逻辑卷大小，`-n` 是逻辑卷名称，`myvg` 是卷组名。

------

## ✅ 步骤四：格式化逻辑卷

格式化新创建的逻辑卷为 ext4（或其他文件系统）：

```
sudo mkfs.ext4 /dev/myvg/home_lv
```

------

## ✅ 步骤五：挂载逻辑卷到临时目录

为了迁移 `/home` 数据，我们需要先挂载逻辑卷到一个临时位置，比如 `/mnt/home`：

```
sudo mkdir /mnt/home
sudo mount /dev/myvg/home_lv /mnt/home
```

------

## ✅ 步骤六：复制 `/home` 数据

将原有 `/home` 中的数据复制到新的逻辑卷中：

```
sudo rsync -av /home/ /mnt/home/
```

> 这会把所有数据复制过去，包括权限、符号链接等。

------

## ✅ 步骤七：备份并修改 `/etc/fstab`

1. 备份原来的 `/etc/fstab`：

   ```
   sudo cp /etc/fstab /etc/fstab.bak
   ```

2. 编辑 `/etc/fstab`，将原来的 `/home` 挂载点替换为新的逻辑卷：

   ```
   sudo nano /etc/fstab
   ```

   找到类似下面的行（根据你的实际情况修改）：

   ```
   /dev/sda2       /home           ext4    defaults        0 2
   ```

   替换为：

   ```
   /dev/myvg/home_lv  /home           ext4    defaults        0 2
   ```

3. 保存并退出编辑器。

------

## ✅ 步骤八：卸载并重新挂载 `/home`

现在我们卸载原来的 `/home` 并重新挂载新的逻辑卷：

```
sudo umount /home
sudo mount /dev/myvg/home_lv /home
```

------

## ✅ 步骤九：验证挂载是否成功

```
df -h | grep '/home'
```

你应该看到类似如下输出：

```
/dev/mapper/myvg-home_lv  50G  2.3G  47G   5% /home
```

------

## ✅ 步骤十：重启测试（可选）

你可以重启系统，确认 `/home` 是否正常挂载：

```
sudo reboot
```

------

## ✅ 后续扩容操作（如果需要）

如果你以后还需要扩容 `/home`，可以这样做：

1. 扩展逻辑卷：

   ```
   sudo lvextend -L +20G /dev/myvg/home_lv
   ```

2. 扩展文件系统（以 ext4 为例）：

   ```
   sudo resize2fs /dev/myvg/home_lv
   ```

   如果是 xfs 文件系统：

   ```
   sudo xfs_growfs /dev/myvg/home_lv
   ```

------

## ✅ 总结

| 步骤       | 说明                                         |
| :--------- | :------------------------------------------- |
| 创建 PV    | 将磁盘或分区加入 LVM                         |
| 创建 VG    | 组合多个 PV 成一个卷组                       |
| 创建 LV    | 在卷组中创建逻辑卷                           |
| 格式化     | 对逻辑卷进行文件系统格式化                   |
| 迁移数据   | 使用 `rsync` 将旧 `/home` 数据迁移到新逻辑卷 |
| 修改 fstab | 更新挂载配置                                 |
| 重新挂载   | 卸载旧挂载，挂载新逻辑卷                     |
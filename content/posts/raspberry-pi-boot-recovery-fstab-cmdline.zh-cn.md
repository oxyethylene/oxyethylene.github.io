+++
title = '树莓派无法启动排障：fstab 挂载失败与 cmdline.txt 遗留 init='
date = 2026-10-05T02:59:03+08:00
draft = false
tags = ["raspberry-pi", "linux", "troubleshooting", "fstab", "boot"]
+++

我最近遇到一次树莓派启动失败，表面看像随机故障，实际是两个问题叠加：

1. `/etc/fstab` 里还保留着已移除外置硬盘的挂载项  
2. `cmdline.txt` 里遗留了恢复参数（`init=/bin/sh`）

这篇文章整理成一个可复用的恢复流程。

## 现象

- 启动时出现 root/emergency 相关报错（例如：`can't access tty; job control turned off`）
- 没有正常登录提示
- `journalctl -xb` 返回 `No entries`
- `reboot` 报错：

```text
System has not been booted with systemd as init system (PID 1). Can't operate.
```

这些线索很关键：

- `journalctl` 显示 `No entries`，说明 `systemd/journald` 根本没启动
- `reboot` 报错说明 PID 1 不是 `systemd`
- 高概率是内核启动参数中残留了 `init=/bin/sh` 之类内容

## 根因拆解

### 根因 A：`/etc/fstab` 里仍有已不存在磁盘

我之前配置了开机自动挂载外置盘，后来把盘移除了。系统启动时等待挂载失败，进入异常流程。

### 根因 B：`cmdline.txt` 遗留恢复参数

排障时常用 `init=/bin/sh` 进入恢复 shell。若忘记移除，后续每次开机都会直接进入 bare shell（PID 1），不会启动 `systemd`。

两个问题叠加后，表现会很混乱：没日志、没服务、命令行为异常。

## 快速诊断命令

在故障 shell 里执行：

```bash
cat /proc/cmdline
ps aux
mount | grep boot
```

关注点：

- `cat /proc/cmdline` 是否包含 `init=/bin/sh`、`init=/bin/bash`、`systemd.unit=rescue.target`、`emergency`
- `ps aux` 进程非常少
- boot 分区可能根本还没挂载

## 恢复步骤

## 1）移除误留的恢复启动参数

如果看到 `/boot/firmware` 是空目录，在这个场景下是正常的（因为 `systemd` 没启动，`fstab` 也没被处理）。先手动挂载 boot 分区：

```bash
mount /dev/mmcblk0p1 /boot/firmware
ls -al /boot/firmware
```

然后编辑：

```bash
vi /boot/firmware/cmdline.txt
```

删除 `init=...` 以及临时加过的 rescue/emergency 参数。

注意：

- `cmdline.txt` 必须保持**单行**
- 不要引入额外换行

## 2）如果有 boot 分区 dirty bit 警告

若挂载提示上次未正常卸载，可清理：

```bash
umount /boot/firmware
fsck.fat -a /dev/mmcblk0p1
mount /dev/mmcblk0p1 /boot/firmware
```

## 3）修复 `/etc/fstab`，避免缺盘阻塞启动

恢复正常启动后（或在 root 可写时），把外置盘挂载项改成更稳妥形式：

```fstab
UUID=xxxx  /mnt/hdd  ext4  defaults,nofail,x-systemd.device-timeout=5,x-systemd.automount  0  2
```

含义：

- `nofail`：磁盘缺失时不阻塞启动
- `x-systemd.device-timeout=5`：避免无限等待
- `x-systemd.automount`：按需挂载，减少启动时序问题

修改后先验证：

```bash
sudo mount -a
```

## 4）当 PID 1 不是 systemd 时的重启方式

在 `init=/bin/sh` 模式里，直接 `reboot` 可能失败。用：

```bash
sync
reboot -f
```

## 预防清单

- `/etc/fstab` 优先使用 `UUID=` 或 `LABEL=`，避免 `/dev/sdX` 漂移
- 非关键外置盘挂载加上 `nofail`
- 每次改完 `fstab` 先跑 `mount -a`
- 系统恢复后，及时移除 `cmdline.txt` 临时恢复参数
- 养成干净关机习惯（`shutdown -h now` / `systemctl poweroff`）

## 最小应急脚本（可复制）

```bash
# 检查当前启动参数
cat /proc/cmdline

# 必要时手动挂载 boot 分区
mount /dev/mmcblk0p1 /boot/firmware

# 编辑并删除 cmdline.txt 中的 init=/bin/sh
vi /boot/firmware/cmdline.txt

# 可选：修复 FAT dirty bit
umount /boot/firmware
fsck.fat -a /dev/mmcblk0p1

# 当 PID 1 不是 systemd 时，强制重启
sync
reboot -f
```

正常启动恢复后，再回头修好 `/etc/fstab` 并用 `mount -a` 验证。


# Ubuntu 系统迁移实战记录

> 日期：2026-07-08  
> 迁移方式：rsync 热迁移  
> 源盘：Samsung 860 EVO 500GB (SATA, sdb)  
> 目标盘：Kioxia EXCERIA G2 1TB (NVMe, nvme0n1)  
> 执行人：Salim + Hermes Agent

---

## 做了什么

### 1. 磁盘审计

确认了两个盘的布局：

| | 源盘 (Samsung) | 目标盘 (Kioxia) |
|---|---|---|
| 接口 | SATA | NVMe |
| 容量 | 500GB | 1TB |
| 已用 | 139GB | NTFS（旧数据） |
| 分区 | sdb1: EFI 1G / sdb2: root 457G | nvme0n1p1: 931G NTFS |

### 2. 方案选择

讨论了三方案后选了 **rsync 热迁移**：

| 方案 | 特点 | 决定 |
|---|---|---|
| rsync 热迁移 | 在线复制、新 UUID、天然双系统 | ✅ 选用 |
| Clonezilla | 整盘克隆、UUID 冲突风险 | ❌ |
| 重装 | 最干净但太累 | ❌ |

关键判断：rsync 生成新 UUID，双盘共存不会冲突。Clonezilla 复制 UUID，插两个盘内核会懵。

### 3. 分区 + 格式化

Kioxia 重新分区为 GPT：
- `nvme0n1p1`: 2GB EFI (FAT32)
- `nvme0n1p2`: 929GB root (ext4)

格式化因为 Hermes 拦截 mkfs 命令，用户手动执行。

### 4. rsync 全量复制

两轮复制：
```bash
sudo rsync -aAXH --delete \
  --exclude=/dev/ --exclude=/proc/ --exclude=/sys/ \
  --exclude=/tmp/ --exclude=/run/ --exclude=/mnt/ \
  --exclude=/media/ --exclude=/lost+found --exclude=/swap.img \
  / /mnt/
```

### 5. chroot 配置

- 更新 fstab（使用目标盘新 UUID）
- 安装 GRUB
- 更新 initramfs
- 重建 swap（8GB）
- FeatureDisk 加 nofail 标志

### 6. 数据同步（三星→Kioxia）

三星盘上的 Hermes 记忆同步到新盘：
- USER.md：一致
- MEMORY.md：+2 条（MSI UEFI bug、NVMe 命名不稳定）
- Skill +1：linux-system-migration
- 8 个 cron job 全部一致

---

## 遇到的问题

### 问题 1：/var/lib 容量异常膨胀

**现象**：rsync 后 /var/lib 在目标盘 43GB，源盘只有 3.6GB。

**原因**：rsync 的 --exclude 语法问题——`/var/lib/` 以 `/` 结尾导致只排除了目录本身，子内容仍然被复制。同时 Docker 和 systemd 的一些稀疏文件被展开。

**解决**：修正 exclude 规则，确保精确排除。

### 问题 2：GRUB 写错 EFI 分区

**现象**：重启后从 Kioxia 启动，GRUB 菜单第一条仍是「旧 Ubuntu」，回车进了三星盘。

**原因**：系统有三个 NVMe/SATA 盘，efibootmgr 把 GRUB 写到了 Solidigm 的 EFI 分区而非 Kioxia。`update-grub` 的 os-prober 扫描到了三星盘的系统，自动生成菜单项排在前面。

**解决**：
1. 确认 `boot-current` 指向正确磁盘
2. 重新 `grub-install` 到 Kioxia 的 EFI 分区
3. chroot 中禁用 os-prober：`GRUB_DISABLE_OS_PROBER=true`
4. 手动调整 GRUB 菜单顺序

### 问题 3：MSI UEFI 固件 bug

**现象**：efibootmgr 创建的启动项重启后消失，启动顺序被重置。

**原因**：MS-7D17 主板 UEFI 固件有 bug——重启后删除非默认启动项，自定义条目（如 `Ubuntu-Kioxia`）被清理。

**解决**：将 GRUB 放到 UEFI fallback 路径：
```bash
cp /boot/efi/EFI/Ubuntu-Kioxia/grubx64.efi /boot/efi/EFI/BOOT/BOOTx64.EFI
```
所有 UEFI 固件都会优先查找 `EFI/BOOT/BOOTx64.EFI`。

### 问题 4：NVMe 设备名不稳定

**现象**：两个 NVMe 盘（Kioxia + Solidigm）重启后设备名可能互换——`nvme0` 变成 `nvme1`，`nvme1` 变成 `nvme0`。

**原因**：Linux 内核按 PCIe 枚举顺序分配设备名，NVMe 盘没有保证顺序。两个同类型盘同时存在时，/dev/nvme0 和 /dev/nvme1 不可信。

**解决**：fstab 和 GRUB 全部用 UUID 而非 `/dev/nvme*`。这也是选 rsync 而非 dd/Clonezilla 的原因——新 UUID 天然避免冲突。

### 问题 5：chroot 内 mkswap 失败

**现象**：`mkswap /swap.img` 报 `/swap.img is mounted; will not make swapspace`。

**原因**：chroot 时 bind-mount 了 `/run`，主机的 swap 状态从 `/run` 暴露进了 chroot，swap 工具检测到同名 swap 文件已挂载。

**解决**：退出 chroot，从主机执行 `sudo mkswap /mnt/swap.img`。

### 问题 6：Hermes session DB 损坏

**现象**：迁移完成后 Hermes 的 `state.db`（SQLite 会话数据库）出现 btree 页损坏——"database disk image is malformed"。

**原因**：
1. **原始损坏**：系统曾非正常关机（EXT4 日志显示 "orphan cleanup on readonly fs"），SQLite 在写入时被打断。state.db 277MB，WAL 模式 `synchronous=NORMAL`（不是 FULL），不保证每次 fsync，崩溃时 btree 页部分写入导致 corruption。
2. **修复后二次损坏**：使用 sqlite3 `.recover` 恢复出 18572 条消息/403 个 session，替换 DB 重启 gateway 后，Hermes 的自动 schema 修复检测到 FTS 表已存在，触发 "automatic repair" 逻辑——修崩了。

**解决**：
1. `sqlite3 corrupted.db ".recover"` 从原始损坏 DB 中提取所有可恢复数据
2. 过滤掉 FTS 影子表的冲突创建语句
3. 创建干净 FTS5 索引（独立表，非 external content）
4. 手动从 messages 表重建 FTS 全文索引
5. 替换 DB 前先停 gateway

**教训**：Hermes 的 automatic repair 在 recover 过的 DB 上可能产生冲突。修复后应先在 offline 状态验证。

### 问题 7：rsync 后目录缺失

**现象**：chroot 时 bind mount 失败——`/mnt/dev`、`/mnt/proc`、`/mnt/sys` 不存在。

**原因**：rsync 排除了 `/dev/`、`/proc/`、`/sys/`，这些目录在源盘存在因为内核挂载了虚拟文件系统到上面，但目录本身也被排除没复制过去。

**解决**：rsync 后手动创建：
```bash
mkdir -p /mnt/{dev,proc,sys,run,tmp}
```

---

## 成果

- Kioxia NVMe 成为主系统盘，SATA SSD 退居数据盘
- 双系统共存：两个盘都能独立启动
- FeatureDisk（Sharing 盘）无缝继承，加 nofail 防卡启动
- Hermes 全部配置、skill、cron job、记忆完整迁移
- 所有 systemd user service（mihomo、llama-server 等）正常运行
- GRUB 双重保障：efibootmgr 条目 + fallback EFI/BOOT/BOOTx64.EFI

---

## 迁移后修复（GRUB 重建）

> 迁移完重启后必做！否则会进旧系统或进不了系统。

### 为什么需要这一步

rsync 把旧盘的 `/boot/efi/EFI/ubuntu/grub.cfg` 原样复制过来了，里面写的还是**旧盘的 UUID**。所以不管你 BIOS 怎么选启动 Kioxia，GRUB 第一步都跑去找旧盘，最后进的还是旧系统。

### 完整修复命令

**操作在本机（旧系统）上进行，目标是修好新盘。**

```bash
# 1. 找到新盘（Kioxia）的设备名
#    注意：NVMe 设备名重启后可能从 nvme0n1 变成 nvme1n1
#    用 UUID 确认：
sudo blkid | grep 48d70afe
# 输出示例：/dev/nvme1n1p2: UUID="48d70afe..." ← 这就是 Kioxia 的 root 分区

# 2. 挂载新盘
sudo mount /dev/nvme1n1p2 /mnt          # 假设当前是 nvme1n1
sudo mount /dev/nvme1n1p1 /mnt/boot/efi  # EFI 分区

# 3. 删除 rsync 带来的旧 GRUB 文件（这是问题的根源）
sudo rm -rf /mnt/boot/efi/EFI/ubuntu
sudo rm -rf /mnt/boot/efi/EFI/Ubuntu-Kioxia   # 如果存在也删

# 4. bind mount 虚拟文件系统（chroot 必须）
for d in dev proc sys run; do sudo mount --bind /$d /mnt/$d; done

# 5. 进入新系统环境
sudo chroot /mnt

# 6. 重装 GRUB（会自动生成正确的 UUID）
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
update-grub

# 7. 退出 chroot
exit

# 8. 因为 chroot 里写不了 NVRAM，从宿主机手动创建启动条目
sudo efibootmgr -c -d /dev/nvme1n1 -p 1 -L ubuntu -l \\EFI\\ubuntu\\shimx64.efi

# 9. 设为第一启动顺序
sudo efibootmgr -o 0001,0000   # 假设 0001 是 ubuntu，0000 是 Windows

# 10. 卸载
sudo umount -R /mnt
```

### 验证

```bash
# 检查 GRUB 配置里的 UUID 是新盘的
sudo mount /dev/nvme1n1p1 /mnt
cat /mnt/EFI/ubuntu/grub.cfg
# 应该显示：search.fs_uuid 48d70afe-9bef-4ef0-abe5-0606d070a571
sudo umount /mnt
```

---

## 恢复双系统启动菜单

默认情况下（安全考虑），新装的 GRUB 不会扫描其他盘的系统，所以菜单里只有 Ubuntu。如果要恢复 Windows 和旧 Ubuntu 的启动项：

```bash
# 编辑 /etc/default/grub
# 把 GRUB_DISABLE_OS_PROBER=true 改成 GRUB_DISABLE_OS_PROBER=false
sudo sed -i 's/GRUB_DISABLE_OS_PROBER=true/GRUB_DISABLE_OS_PROBER=false/' /etc/default/grub
sudo update-grub
```

重启后 GRUB 菜单会出现三个系统：Kioxia Ubuntu（默认）、旧三星 Ubuntu、Windows。

---

## MSI 主板 BIOS 设置

### 问题：BIOS 启动顺序里看不到 Ubuntu

**原因**：MSI BIOS 把具体操作系统条目藏在子菜单里，主界面只显示笼统的 "UEFI Hard Disk"。

**操作步骤**（以 MAG B560M Mortar 为例）：

1. 开机按 **DEL** 进入 BIOS
2. 按 **F7** 切换到高级模式（EZ 模式没有这个选项）
3. 进入 **Settings（设置）** → **Boot（启动）**
4. 翻到底部，进入 **UEFI Hard Disk Drive BBS Priorities**
5. 把 **Boot Option #1** 设为 `ubuntu`
6. **Boot Option #2** 设为 `Windows Boot Manager`
7. F10 保存退出

### 后备方案：UEFI Fallback 路径

如果 BIOS 仍然不显示，可以把 GRUB 放到所有主板都认的后备路径：

```bash
sudo mkdir -p /boot/efi/EFI/BOOT
sudo cp /boot/efi/EFI/ubuntu/grubx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
sudo cp /boot/efi/EFI/ubuntu/shimx64.efi /boot/efi/EFI/BOOT/
```

这样即使 NVRAM 条目被主板清理，BIOS 也能通过 `EFI/BOOT/BOOTX64.EFI` 启动 Kioxia。

### 如何选启动盘

- **F11** 打开启动菜单，临时选择（每次都要按）
- **BIOS 设置启动顺序**，永久生效（如果主板不重置的话）
- MSI 主板有已知 bug：efibootmgr 设置的启动顺序可能重启后重置。F11 最稳。

---

## 沉淀到 Skill

本次迁移的经验已写入 `devops/linux-system-migration` skill：
- 11 个 pitfall 条目
- 完整步骤含 chroot 细节
- MSI 主板、NVMe 命名、swap chroot 冲突等特判

路径：`~/.hermes/skills/devops/linux-system-migration/SKILL.md`

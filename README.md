# Mi Max 2 (oxygen) DroidSpaces 内核构建包

基于 halibw 的 [android_kernel_xiaomi_oxygen](https://github.com/halibw/android_kernel_xiaomi_oxygen) `lineage-21` 分支（4.19.313，LOS 21 官方发布版同款源码），追加 DroidSpaces 所需的 namespace/cgroup 配置后用 GitHub Actions 云端编译。

## 已完成的本地验证（macOS kconfig 试跑）

`droidspaces.config` 追加到 `vendor/msm8953-perf_defconfig` 后 olddefconfig 合成，关键选项全部生效：

| 类别 | 选项（全部 =y） |
|---|---|
| namespace | NAMESPACES, PID_NS, UTS_NS, IPC_NS, USER_NS, NET_NS |
| IPC | SYSVIPC, POSIX_MQUEUE |
| 设备/文件 | DEVTMPFS, OVERLAY_FS, TMPFS_XATTR, FHANDLE |
| cgroup | CGROUP_PIDS, CGROUP_DEVICE, CGROUP_SCHED, FAIR_GROUP_SCHED, CFS_BANDWIDTH, MEMCG |
| 网络 | VETH, BRIDGE, BRIDGE_NETFILTER, NF_CONNTRACK, NF_NAT, MASQUERADE, MACVLAN, NF_TABLES |
| 其他 | SECCOMP_FILTER, TMPFS_POSIX_ACL |

保持不变（ROM 兼容性关键项）：`SCHED_WALT=y`（调度器不动）、`MODULE_SIG_FORCE=y`（本构建自动签名 rdbg.ko）、`LOCALVERSION="-perf-droidspaces"`（刷入后 `/proc/version` 可直接辨认）。

补丁：Droidspaces 官方 non-GKI 两枚（xt_qtaguid panic 修复 + cgroup prefix 修复），workflow 内按条件应用。

## 使用步骤（GitHub Actions 编译）

1. 在 GitHub 新建仓库（建议名 `oxygen-droidspaces-kernel`，Private 即可）
2. 推送本目录：

```bash
cd /Users/alex/work/rootpad/oxygen-kernel-build
git init -b main
git add -A
git commit -m "oxygen 4.19 kernel with DroidSpaces"
git remote add origin https://github.com/<你的用户名>/oxygen-droidspaces-kernel.git
git push -u origin main
```

3. 打开仓库 → **Actions** 页签 → 左侧选 "Build oxygen kernel 4.19 with DroidSpaces" → **Run workflow**
   - 若下拉里没有该 workflow，等 1 分钟刷新一次（首次 push 后 GitHub 需要注册 workflow）
4. 构建约 20-30 分钟。产物 artifact 名为 `oxygen-droidspaces-kernel`，包含：
   - `Image.gz-dtb`（内核+DTB，刷 boot.img 用）
   - `AnyKernel3 zip`（备用，需 TWRP 才能刷）
   - `modules/`（若产出 .ko）
   - `config.txt` / `build.log` / `System.map`

## 编译成功后的刷机流程（已获用户授权）

ROM：`lineage-21.0-20240216-UNOFFICIAL-oxygen.zip`（已在 `/Users/alex/work/rootpad/oxygen-rom/`，827M）

1. **从 ROM zip 提取 boot.img**（payload.bin → payload-dumper 工具）
2. `fastboot flash recovery recovery-los21.img` + `fastboot wipe-super super_empty.img`（retrofit dynamic 分区重置）
3. 进 LOS recovery → **Format data/factory reset（清数据，已授权）** → `adb sideload` ROM zip
4. 重启前回 bootloader：Magisk app 给 ROM 原版 boot.img 打补丁 → `fastboot flash boot magisk_patched.img`（先恢复 root）
5. 进系统后（root 可用）：设备上用 `/data/adb/magisk/magiskboot` 把 CI 产出的 `Image.gz-dtb` 替换进 boot.img，回刷 boot 分区
6. 验证：
   - `cat /proc/version` → 应显示 `4.19.313-perf-droidspaces`
   - `ls /proc/self/ns/` → 应出现 `pid uts ipc user net mnt cgroup`
   - DroidSpaces app → 不再报"内核不支持"，创建容器实测

## 回滚

- 内核：重刷 Magisk patched 的 ROM 原版 boot.img（备份在 oxygen-backup/）
- 整个 ROM：重刷当前 5.4 WIP ROM（用户自行持有）或 oxygen-backup/boot-orig.img 对应的原厂系统

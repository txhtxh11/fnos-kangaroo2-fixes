# Kangaroo 二代 飞牛(fnOS) 重启卡死 & 内置电池 UPS 识别修复

适用于 **Foxconn Kangaroo（Intel Atom x5-Z8500 / Cherry Trail 平台）** 上运行的飞牛 OS。

两个问题是**独立**的，可以分开修，也可以一起修。**但一起修时有顺序要求**（见下）。

---

## 包含内容

```
kangaroo二代飞牛重启及ups识别修复/
├── README.md                      ← 本文件
├── skills/                        ← 给 AI Agent 用的技能（OpenClaw 格式）
│   ├── fnos-reboot-pci/SKILL.md         （重启卡死修复）
│   └── fnos-acpi-ups-battery/SKILL.md   （电池 UPS 识别修复）
├── packages/                      ← 飞牛手动安装包（应用中心 → 手动安装）
│   ├── fn-reboot-pci-1.0.0.fpk          （重启卡死修复）
│   └── fn-ups-acpi-1.0.0.fpk            （电池 UPS 识别修复）
└── docs/                          ← 原始修复记录（含完整原理与重做步骤）
    ├── 修复说明.txt
    └── 重启修复说明-reboot_pci.txt
```

---

## 问题一：点重启后卡在重启界面

**症状**：飞牛页面点「重启」后卡住，电源灯常亮，需长按电源键强制关机。

**原因**：Cherry Trail 平台已知固件缺陷，内核执行 ACPI 重启时被卡住。

**修复**：给内核加 `reboot=pci` 参数。

- 手动：见 `skills/fnos-reboot-pci/SKILL.md`
- 安装包：`packages/fn-reboot-pci-1.0.0.fpk`

---

## 问题二：不识别内置电池为 UPS

**症状**：飞牛 UPS 页面空白；`/sys/class/power_supply/` 里没有 `BATC`；`upower -e` 无电池。

**原因**：固件 DSDT 中电池设备 `I2C4.BATC` 的 `_STA` 方法在启动早期访问了
GenericSerialBus 区域 `DVEC`（此时无 handler）而中止，电池被判定为"不存在"。

**修复**：用打补丁的 DSDT 表通过 initrd override 替换固件表。

- 手动：见 `skills/fnos-acpi-ups-battery/SKILL.md`
- 安装包：`packages/fn-ups-acpi-1.0.0.fpk`

---

## ⚠️ 两个一起修复时的顺序（重要）

**必须先做「重启修复」，再做「电池修复」**：

```
1. 先：reboot=pci + update-grub        ← update-grub 会重新生成 /boot/grub/grub.cfg
2. 后：把 acpi_override 插入 initrd 行  ← 必须在 update-grub 之后
```

**反序的后果**：`update-grub` 会把 grub.cfg 整个重新生成，**覆盖掉 ACPI override 那行**，
导致电池修复白做（表现为：`dmesg | grep 'Physical table override'` 没有输出）。

**如果中途又跑了 `update-grub`**：需要重新执行一次电池修复（安装包点一次「启动」，
或手动重做 skill 里的"步骤 7"）。

---

## 安装步骤（用安装包）

1. 把两个 `.fpk` 传到飞牛（文件管理 → 上传）
2. 飞牛 **应用中心 → 手动安装** → 先选 `fn-reboot-pci-1.0.0.fpk`
   - 安装向导里选 `pci`，装完点【启动】
3. 再装 `fn-ups-acpi-1.0.0.fpk`
   - 向导里「自动接入 GRUB」选**是**，装完点【启动】
4. **长按电源键冷启动**（不要用 SSH 的 reboot 命令验证，本机不可靠）

## 验证（冷启动后）

```bash
# 重启修复
cat /proc/cmdline | grep -o reboot=pci

# 电池修复
dmesg | grep 'ACPI: DSDT'                    # 应看到 Physical table override
ls /sys/class/power_supply/                  # 应出现 BATC
upower -e                                    # 应出现 battery_BATC

# 应用自带状态
/var/apps/fn-reboot-pci/cmd/main status
/var/apps/fn-ups-acpi/cmd/main status
```

## 撤销

```bash
# 重启修复
/var/apps/fn-reboot-pci/cmd/main rollback

# 电池修复
/var/apps/fn-ups-acpi/cmd/main rollback
# 然后删除 /boot/acpi_override.cpio 并冷启动
```

两个修复都**不修改系统数据**，仅改动启动参数与注入 ACPI 表，可安全回滚。

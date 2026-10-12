# Kangaroo 二代 飞牛(fnOS) 重启卡死 & 内置电池 UPS 识别修复

适用于 **Foxconn Kangaroo（Intel Atom x5-Z8500 / Cherry Trail 平台）** 上运行的飞牛 OS。

两个问题是**独立**的。可以分开修，也可以一个包装完 —— **二合一版在安装向导里就能分别勾选**。

---

## 两种用法，按需选

### 方式一：二合一（推荐，省事）

- **技能**：`skills/fnos-kangaroo2-fixes/`
- **安装包**：`packages/fn-kangaroo2-fix-1.0.0.fpk`

安装向导里有两个下拉：
- **① 修复「点重启卡在重启界面」** → 做 / 跳过
- **② 修复「不识别内置电池为 UPS」** → 做 / 跳过

两个都选"做"也不会冲突 —— 应用会**自动按正确顺序**执行（先重启、后电池）。

### 方式二：独立包（想单独控制时）

| 修复 | 技能 | 安装包 |
|---|---|---|
| 重启卡死 | `skills/fnos-reboot-pci/` | `packages/fn-reboot-pci-1.0.0.fpk` |
| 电池 UPS | `skills/fnos-acpi-ups-battery/` | `packages/fn-ups-acpi-1.0.0.fpk` |

⚠️ 用**独立包**两个都装时，**必须先装/启动重启修复，再装/启动电池修复**（见下）。

---

## 目录结构

```
kangaroo二代飞牛重启及ups识别修复/
├── README.md
├── skills/
│   ├── fnos-kangaroo2-fixes/SKILL.md    ← 二合一（推荐）
│   ├── fnos-reboot-pci/SKILL.md         ← 独立：重启
│   └── fnos-acpi-ups-battery/SKILL.md   ← 独立：电池
├── packages/
│   ├── fn-kangaroo2-fix-1.0.0.fpk       ← 二合一（推荐）
│   ├── fn-reboot-pci-1.0.0.fpk          ← 独立：重启
│   └── fn-ups-acpi-1.0.0.fpk            ← 独立：电池
└── docs/
    ├── 修复说明.txt
    └── 重启修复说明-reboot_pci.txt
```

---

## 问题一：点重启后卡在重启界面

**症状**：飞牛页面点「重启」后卡住，电源灯常亮，需长按电源键。
**原因**：Cherry Trail 平台已知固件缺陷。
**修复**：内核参数加 `reboot=pci`。

## 问题二：不识别内置电池为 UPS

**症状**：UPS 页面空白；`/sys/class/power_supply/` 无 `BATC`；`upower -e` 无电池。
**原因**：固件 DSDT 中 `I2C4.BATC` 的 `_STA` 启动早期访问 GenericSerialBus 区域 `DVEC`
（此时无 handler）而中止，电池被判为"不存在"。
**修复**：用打补丁的 DSDT 表通过 initrd override 替换固件表。

---

## ⚠️ 顺序要求（仅当用「独立包」两个都做时）

```
1. 先：reboot=pci + update-grub      ← update-grub 会重新生成 /boot/grub/grub.cfg
2. 后：把 acpi_override 插入 initrd 行
```

**反序**：`update-grub` 会覆盖掉 override 行 → 电池修复白做
（症状：`dmesg | grep 'Physical table override'` 无输出）。

> 用**二合一包**则无需操心 —— 它内部已固定顺序。

---

## 安装步骤（二合一包）

1. 把 `fn-kangaroo2-fix-1.0.0.fpk` 传到飞牛（文件管理 → 上传）
2. 飞牛 **应用中心 → 手动安装** → 选它
3. 向导里按需勾选两个修复 → 装完点【启动】
4. **长按电源键冷启动**（不要用 SSH `reboot` 验证，本机不可靠）

## 验证（冷启动后）

```bash
# 重启修复
cat /proc/cmdline | grep -o reboot=pci

# 电池修复
dmesg | grep 'ACPI: DSDT'                    # 应见 Physical table override
ls /sys/class/power_supply/                  # 应出现 BATC
upower -e                                    # 应出现 battery_BATC

# 应用状态
/var/apps/fn-kangaroo2-fix/cmd/main status
```

## 撤销

```bash
/var/apps/fn-kangaroo2-fix/cmd/main rollback
# 独立包则分别执行各自的 rollback
```

两个修复都**不改系统数据**，仅改动启动参数与注入 ACPI 表，可安全回滚。

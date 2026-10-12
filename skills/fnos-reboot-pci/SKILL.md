---
name: fnos-reboot-pci
description: 飞牛(fnOS)点重启后卡在重启界面、需长按电源键，用 reboot=pci 内核参数修复。何时用：fnOS/群晖类 NAS 装在 Intel Atom/Cherry Trail 平台(如 Foxconn Kangaroo)上，页面点重启无效、卡住、电源灯常亮。
---

# fnOS 重启卡死修复（reboot=pci）

## 何时用

- fnOS（或同类 Debian 底 NAS）**点页面重启后卡在重启界面**，电源灯常亮，需长按电源键强制关机
- 已知平台：**Foxconn Kangaroo（Atom x5-Z8500 / Cherry Trail）**
- 原因：内核执行 ACPI 重启时被设备固件缺陷卡住（Cherry Trail 平台已知兼容问题）

## 修复（一条命令）

```bash
sudo cp /etc/default/grub /etc/default/grub.bak && \
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 reboot=pci"/' /etc/default/grub && \
sudo update-grub && \
echo "修复完成，重启后生效"
```

## 验证

```bash
cat /proc/cmdline    # 输出中应包含 reboot=pci
```

## 无效时的备选

依次把 `reboot=pci` 换成下面之一，重跑上面三步：

- `reboot=efi` —— 用 EFI 方式重启
- `reboot=acpi` —— 用 ACPI 方式重启

## 撤销

```bash
sudo cp /etc/default/grub.bak /etc/default/grub && sudo update-grub
```

## ⚠️ 与「电池 UPS 识别修复」一起做时的顺序

若你**同时**要修电池识别（`fnos-acpi-ups-battery` 技能），**本修复必须先做**：

```
1. 先：本修复（reboot=pci + update-grub）  ← update-grub 会重新生成 grub.cfg
2. 后：DSDT override 插入 initrd          ← 必须在 update-grub 之后
```

**反序会让 `update-grub` 覆盖掉 ACPI override 行**，导致电池修复失效（dmesg 里没有
`Physical table override`）。做完本修复后若要补 ACPI override，重新执行它的"步骤 7"即可。

## 说明

- 此参数只改变系统重启方式，不影响其他功能。
- 非 Cherry Trail / Atom 平台无需此参数。
- 重新安装 fnOS 系统后本参数会丢失，需重新执行。

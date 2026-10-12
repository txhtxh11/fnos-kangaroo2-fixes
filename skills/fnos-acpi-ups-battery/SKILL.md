---
name: fnos-acpi-ups-battery
description: 飞牛(fnOS)不识别内置电池为 UPS 时，用 DSDT ACPI override 注入补丁表修复。何时用：fnOS/群晖类 NAS 装在 Intel 平板/迷你机(如 Foxconn Kangaroo、Cherry Trail)上，看不到电池、/sys/class/power_supply 无 BATC、upower 无电池、UPS 页面空白。
---

# fnOS 内置电池 UPS 不识别修复（DSDT override）

## 何时用

- fnOS（或同类 Debian 底 NAS）装在 **Intel 平板/迷你机**上，**看不到内置电池**：
  `/sys/class/power_supply/` 无 `BATC`、飞牛 UPS 页面空白、`upower -e` 无电池
- 已知平台：**Foxconn Kangaroo（Atom x5-Z8500 / Cherry Trail）**，内核 `*-trim`

> 若你的问题是**点重启卡死**（与电池无关），用另一个技能 `fnos-reboot-pci`。

## 先确认是本类问题

```bash
ls /sys/class/power_supply/
cat /sys/bus/acpi/devices/PNP0C0A:*/status   # 期望 31(0x1F)；0=被判定不存在
dmesg | grep -iE 'ACPI Error|No handler|_STA'  # 看是否有 DVEC / _STA 中止
```

出现 `No handler for Region [DVEC]` 或 `PNP0C0A:0x status=0` → 就是本问题。

**原理**：电池设备 `_STA` 在启动早期访问 GenericSerialBus 区域 DVEC（尚无 handler）而中止
（AE_NOT_EXIST），或被 OSID/BDID 版本门槛拦下，于是被判为不存在。用补丁 DSDT 替换固件表即可。

**关键前提（漏了就静默失效）**：内核只在「签名 + OEMID + OEM Table ID 匹配 **且 OEM Revision 更新**」
时才做**物理表替换**，否则只追加（对 DSDT 等于忽略）。

## 步骤

```bash
# 0) 依赖
sudo apt install -y acpica-tools cpio

# 1) 取出固件 DSDT 并反编译
sudo cat /sys/firmware/acpi/tables/DSDT > dsdt.dat
iasl -d dsdt.dat                      # 生成 dsdt.dsl

# 2) 改 I2C4.BATC 的 _STA：整体替换为强制返回存在且启用
#    Method (_STA, 0, NotSerialized) { Return (0x1F) }
#    原方法体若有 DVEC 访问/OSID 门槛，整段换掉；同时清掉因此产生的死代码(CRS4/CRS5)、
#    修 WIDR 错误，否则 iasl 编译不过。

# 3) 提升 OEM Revision（必做；固件为 3 就写 4）
#    DefinitionBlock ("", "DSDT", 2, "FXN PC", "FOXCONN ", 0x00000004)

# 4) 编译
iasl -sa dsdt.dsl                     # 生成 dsdt.aml

# 5) 填充到与原 DSDT 等长，并修正表头 length(offset 4..7) 与 checksum(offset 9)
ORIG=$(stat -c%s dsdt.dat)
python3 - "$ORIG" <<'PY'
import sys
orig=int(sys.argv[1]); b=bytearray(open('dsdt.aml','rb').read())
if len(b)<orig: b+=b'\x00'*(orig-len(b))
b=b[:orig]
b[4:8]=len(b).to_bytes(4,'little')      # 表头 length
b[9]=0
b[9]=(-sum(b))%256                       # 使全部字节和 %256 == 0
open('dsdt.override.aml','wb').write(b)
print('长度',len(b),'校验和',hex(b[9]))
PY

# 6) 打包成「未压缩」cpio（必须未压缩）
mkdir -p /tmp/ovl/kernel/firmware/acpi
cp dsdt.override.aml /tmp/ovl/kernel/firmware/acpi/DSDT.aml
(cd /tmp/ovl && find . -print | cpio -o -H newc) | sudo tee /boot/acpi_override.cpio >/dev/null

# 7) GRUB 把 override 拼到 initrd 列表最前
#    编辑 /boot/grub/grub.cfg 的 FNOS 主菜单项，linux 行之后改为：
#      initrd /boot/acpi_override.cpio /boot/initrd.img-<当前内核>
#    内核版本用 uname -r 取。
```

## 验证（必须冷启动）

SSH `reboot` 在 Cherry Trail 本机不可靠，**长按电源键冷启动**后：

```bash
dmesg | grep 'ACPI: DSDT'                     # 应看到 Physical table override
ls /sys/class/power_supply/                   # 应出现 BATC
cat /sys/bus/acpi/devices/PNP0C0A:*/status    # 31 (0x1F)
upower -e                                     # 应出现 battery_BATC
```

**回滚**：删掉 grub.cfg initrd 行里的 `/boot/acpi_override.cpio`，删除该文件，冷启动即恢复（数据无损）。

## ⚠️ 与「重启卡死修复」一起做时的顺序

若你**同时**要修重启卡死（`fnos-reboot-pci` 技能），**必须先做重启修复、再做本修复**：

```
1. 先：加 reboot=pci + update-grub   ← update-grub 会重新生成 grub.cfg
2. 后：把 acpi_override 插入 initrd 行 ← 必须在 update-grub 之后
```

**反序会让 `update-grub` 覆盖掉 override 行，白做**（且症状是"dmesg 里没有 Physical table override"）。
如果中途又跑了 `update-grub`，需重新执行本技能的"步骤 7"。

## 坑（都实测踩过）

- **OEM Revision 不提升** → 内核只追加不替换，dmesg 无 `Physical table override`。
- **表长度与原表不等** → 替换失败。
- **checksum 没重算** → 表被内核丢弃。
- **cpio 压缩了** → 内核认不出；必须 `cpio -o -H newc`（未压缩）。
- **用 SSH `reboot` 验证** → 不可靠，必须长按电源键**冷启动**。
- **内核升级后** → initrd 文件名变化，需更新 grub 拼接并重新冷启动验证。
- 别把 `PNP0C0A:00`(I2C1)/`:02`(I2C7) 的 `status=0` 当故障：那是占位/别名设备，
  真电池只有 `I2C4/BATC` 一颗，识别它即可满足 UPS 功能。

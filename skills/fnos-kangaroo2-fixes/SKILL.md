---
name: fnos-kangaroo2-fixes
description: Foxconn Kangaroo(Intel Atom/Cherry Trail) 上飞牛(fnOS)的「重启卡死」与「不识别内置电池为UPS」两个修复，二选一或都做。何时用：fnOS/群晖类 NAS 在该平台上点重启卡住、或看不到电池(UPS页面空白、无 BATC)。
---

# Kangaroo 二代 fnOS 修复（重启卡死 + 电池 UPS 识别）

两个问题**互相独立**，可只做一个，也可都做。**都做时顺序有要求**（见下）。

## 先判断你遇到哪个

```bash
# 重启卡死？—— 点页面重启后卡住、电源灯常亮、需长按电源键
# 电池不识别？
ls /sys/class/power_supply/                                  # 无 BATC → 中招
cat /sys/bus/acpi/devices/PNP0C0A:*/status                   # 0 = 被判不存在
dmesg | grep -iE 'No handler for Region \[DVEC\]|_STA'       # 有 DVEC/_STA 中止 → 中招
```

| 症状 | 用哪个修复 |
|---|---|
| 点重启卡住 | **A. reboot=pci** |
| 电池/UPS 不识别 | **B. DSDT override** |
| 两个都有 | **先 A 后 B**（顺序很关键） |

---

## ⚠️ 顺序要求（两个都做时）

```
1. 先做 A：reboot=pci + update-grub      ← update-grub 会重新生成 /boot/grub/grub.cfg
2. 后做 B：把 acpi_override 插入 initrd   ← 必须在 update-grub 之后
```

**反序的后果**：`update-grub` 会把 grub.cfg 整个重生成，**覆盖掉 B 的 override 行**，
B 白做（症状：`dmesg | grep 'Physical table override'` 无输出）。
若中途又跑了 `update-grub`，需重新执行 B 的"步骤 7"。

---

## A. 重启卡死：加 reboot=pci

```bash
sudo cp /etc/default/grub /etc/default/grub.bak && \
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 reboot=pci"/' /etc/default/grub && \
sudo update-grub
```

验证：`cat /proc/cmdline` 含 `reboot=pci`。
无效时依次试 `reboot=efi` / `reboot=acpi`。
撤销：`sudo cp /etc/default/grub.bak /etc/default/grub && sudo update-grub`。

## B. 电池不识别：DSDT override 注入

原理：电池设备 `_STA` 在启动早期访问 GenericSerialBus 区域 DVEC（尚无 handler）而中止
（AE_NOT_EXIST），或被 OSID/BDID 门槛拦下，被判为不存在。用补丁 DSDT 替换固件表。

**关键前提（漏了就静默失效）**：内核只在「签名 + OEMID + OEM Table ID 匹配 **且 OEM Revision
更新**」时才做**物理表替换**，否则只追加（对 DSDT 等于忽略）。

```bash
# 0) 依赖
sudo apt install -y acpica-tools cpio

# 1) 取出固件 DSDT 并反编译
sudo cat /sys/firmware/acpi/tables/DSDT > dsdt.dat
iasl -d dsdt.dat

# 2) 改 I2C4.BATC 的 _STA，整体替换为：
#    Method (_STA, 0, NotSerialized) { Return (0x1F) }
#    （原体若有 DVEC 访问/OSID 门槛，整段换；清掉因此产生的死代码 CRS4/CRS5、修 WIDR）

# 3) 提升 OEM Revision（固件为 3 就写 4）
#    DefinitionBlock ("", "DSDT", 2, "FXN PC", "FOXCONN ", 0x00000004)

# 4) 编译
iasl -sa dsdt.dsl

# 5) 填充到与原表等长，修正表头 length(offset 4..7) 与 checksum(offset 9)
ORIG=$(stat -c%s dsdt.dat)
python3 - "$ORIG" <<'PY'
import sys
orig=int(sys.argv[1]); b=bytearray(open('dsdt.aml','rb').read())
if len(b)<orig: b+=b'\x00'*(orig-len(b))
b=b[:orig]
b[4:8]=len(b).to_bytes(4,'little')
b[9]=0
b[9]=(-sum(b))%256
open('dsdt.override.aml','wb').write(b)
print('长度',len(b),'校验和',hex(b[9]))
PY

# 6) 打包「未压缩」cpio（必须未压缩）
mkdir -p /tmp/ovl/kernel/firmware/acpi
cp dsdt.override.aml /tmp/ovl/kernel/firmware/acpi/DSDT.aml
(cd /tmp/ovl && find . -print | cpio -o -H newc) | sudo tee /boot/acpi_override.cpio >/dev/null

# 7) GRUB 把 override 拼到 initrd 列表最前（★必须在 A 的 update-grub 之后做）
#    /boot/grub/grub.cfg 的 FNOS 主菜单项：
#      initrd /boot/acpi_override.cpio /boot/initrd.img-<当前内核>
```

验证（**长按电源键冷启动**后；SSH `reboot` 在本机不可靠）：
```bash
dmesg | grep 'ACPI: DSDT'                   # 应见 Physical table override
ls /sys/class/power_supply/                 # 应出现 BATC
upower -e                                   # 应出现 battery_BATC
```
回滚：删掉 grub.cfg initrd 行里的 `/boot/acpi_override.cpio`，删该文件，冷启动即恢复。

---

## 坑（都实测踩过）

- **OEM Revision 不提升** → 内核只追加不替换，无 `Physical table override`。
- **表长度与原表不等 / checksum 没重算** → 替换失败或被丢弃。
- **cpio 压缩了** → 内核认不出；必须 `cpio -o -H newc`。
- **顺序反了** → update-grub 覆盖 override（见上）。
- **用 SSH `reboot` 验证** → 不可靠，必须长按电源键冷启动。
- **内核升级后** → initrd 文件名变化，需更新 grub 拼接并重新冷启动验证。
- 别把 `PNP0C0A:00`(I2C1)/`:02`(I2C7) 的 `status=0` 当故障：那是占位/别名，真电池只有 `I2C4/BATC`。

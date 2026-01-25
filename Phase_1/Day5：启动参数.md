## 学习目标

- 使用 emulator 或真机查看启动参数
  - `/proc/cmdline`
- 将参数与启动阶段进行对应

---

## 查看真机启动参数

```
console=tty0
console=ttyS0,921600n1
root=/dev/ram
vmalloc=400M
slub_debug=OFZPU
swiotlb=noforce
firmware_class.path=/vendor/firmware
page_owner=on
loop.max_part=7
has_battery_removed=0
androidboot.boot_devices=bootdevice,11230000.mmc
phx_rus_conf.main_on=1
phx_rus_conf.recovery_method=2
phx_rus_conf.kernel_time=240
phx_rus_conf.android_time=250
fd_passwd=43555
root=/dev/ram
androidboot.verifiedbootstate=orange
bootopt=64S3,32N2,64N2
buildvariant=user
oppo_boot_mode=0
lcm=1-nt36672c_fhdp_dsi_vdo_dphy_jdi_lcm_drv
short_c_battery_status=0
short_c_switch_status=0
short_c_feature_sw_status=1
short_c_feature_hw_status=0
androidboot.atm=disabled
androidboot.meta_log_disable=0
printk.disable_uart=1
bootprof.pl_t=2779
bootprof.lk_t=8138
bootprof.logo_t=1224
androidboot.serialno=U8L7CQZHNJ59EMTG
androidboot.bootreason=reboot
gpt=1
usb2jtag_mode=0
mrdump_ddrsv=yes
mrdump_cb=0x11e000,0x2000
androidboot.dtb_idx=0
androidboot.dtbo_idx=0
```

这个是我 realme V3 手机的启动参数，下面来解析一下看看，虽然多但是不难

```text
cgroup_disable=pressure
log_buf_len=1M
earlycon=msm_geni_serial,0x98c000
rcupdate.rcu_expedited=1
rcu_nocbs=0-7
console=ttyMSM0,115200n8
androidboot.hardware=qcom
androidboot.console=ttyMSM0
androidboot.memcg=1
lpm_levels.sleep_disabled=1
video=vfb:640x400,bpp=32,memsize=3072000
msm_rtb.filter=0x237
service_locator.enable=1
androidboot.usbcontroller=a600000.dwc3
swiotlb=0
loop.max_part=7
cgroup.memory=nokmem,nosocket
pcie_ports=compat
loop.max_part=7
iptable_raw.raw_before_defrag=1
ip6table_raw.raw_before_defrag=1
kpti=off
iptable_raw.raw_before_defrag=1
buildvariant=user
lk_version=V1.0
androidboot.startupmode=usb_charger
androidboot.mode=normal
printk.disable_uart=1
short_c_feature_sw_status=0
phx_rus_conf.main_on=1
phx_rus_conf.recovery_method=2
phx_rus_conf.kernel_time=240
phx_rus_conf.android_time=250
smooth_soc_switch=0
simcardnum.doublesim=1
saupwk.en=1
androidboot.prjname=19825
esim.status=0
androidboot.verifiedbootstate=orange
androidboot.keymaster=1
androidboot.vbmeta.device=PARTUUID=632ce6f7-bd56-d160-7c61-751eef29ed36
androidboot.vbmeta.avb_version=1.0
androidboot.vbmeta.device_state=unlocked
androidboot.vbmeta.hash_alg=sha256
androidboot.vbmeta.size=19840
androidboot.vbmeta.digest=461cc5132f121232e1ad0056e287f831dfa1c0e55dde53ee8da0f1a0454f6ce1
androidboot.vbmeta.invalidate_on_error=yes
androidboot.veritymode=enforcing
androidboot.bootdevice=1d84000.ufshc
androidboot.fstab_suffix=default
androidboot.boot_devices=soc/1d84000.ufshc
androidboot.serialno=8abb982a
androidboot.mode=charger
androidboot.baseband=msm
msm_drm.dsi_display0=qcom,mdss_dsi_samsung_amb655x_dsc_cmd:
oplus_bsp_tp_custom.dsi_display0=qcom,mdss_dsi_samsung_amb655x_dsc_cmd:sec-s6sy792
androidboot.slot_suffix=_b
rootwait
ro
init=/init 
androidboot.dtbo_idx=0
androidboot.dtb_idx=0
androidboot.force_normal_boot=1
androidboot.mode=normal
androidboot.project_codename=lemonade
androidboot.hw_version=22
androidboot.rf_version=11
androidboot.prj_version=12
androidboot.platform_id=415
androidboot.platform_name=SM8350
androidboot.startupmode=usb_charger
androidboot.enable_dm_verity=1
androidboot.at_location=factory
androidboot.power_cut_test=0
androidboot.secboot=enabled
androidboot.battery.absent=false
androidboot.rpmb_enable=true
androidboot.type=nonrelease
androidboot.cust=0
androidboot.prmec=true
androidboot.opcarrier=none
androidboot.bootcount=0
```

这个是我一加 9 的启动参数，感觉更多了点数据吧，不过也可以等需要了再看，里面一个比较重要的参数是
`init=/init` 这个就是内核初始化里面`__setup("init=", init_setup);`传递的参数，然后执行 `run_init_process(execute_command)`函数

### 1. 控制台与输出相关

```text
console=tty0
console=ttyS0,921600n1
printk.disable_uart=1
```

- **console=tty0**：内核日志输出到第一个虚拟终端（屏幕）。
- **console=ttyS0,921600n1**：内核日志输出到串口 `ttyS0`，波特率 921600，`n1` 表示无校验、1 个停止位。
- **printk.disable_uart=1**：禁止通过 UART 打印内核日志。

---

### 2. 根文件系统与内存

```text
root=/dev/ram
vmalloc=400M
loop.max_part=7
```

- **root=/dev/ram**：使用 RAM 作为初始根文件系统。
- **vmalloc=400M**：为 vmalloc 虚拟内存分配 400MB。
- **loop.max_part=7**：loop 设备最多允许 7 个分区。

---

### 3. 内存分配与调试

```text
slub_debug=OFZPU
swiotlb=noforce
page_owner=on
```

- **slub_debug=OFZPU**：SLUB 分配器开启调试模式。
- **swiotlb=noforce**：使用 SWIOTLB 缓冲区，`noforce` 表示不强制开启。
- **page_owner=on**：跟踪每个内存页的所有者，用于调试。

---

### 4. 固件和设备相关

```text
firmware_class.path=/vendor/firmware
androidboot.boot_devices=bootdevice,11230000.mmc
androidboot.serialno=U8L7CQZHNJ59EMTG
lcm=1-nt36672c_fhdp_dsi_vdo_dphy_jdi_lcm_drv
usb2jtag_mode=0
```

- **firmware_class.path**：固件加载路径。
- **androidboot.boot_devices**：启动设备信息（eMMC）。
- **androidboot.serialno**：设备序列号。
- **lcm**：LCD 模组驱动信息。
- **usb2jtag_mode**：USB 转 JTAG 模式开关。

---

### 5. 电池与电源

```text
has_battery_removed=0
short_c_battery_status=0
short_c_switch_status=0
short_c_feature_sw_status=1
short_c_feature_hw_status=0
```

- **has_battery_removed=0**：设备有电池。
- **short*c*\***：短路保护相关状态。

---

### 6. Android 系统特定参数

```text
androidboot.verifiedbootstate=orange
bootopt=64S3,32N2,64N2
buildvariant=user
oppo_boot_mode=0
androidboot.bootreason=reboot
androidboot.atm=disabled
androidboot.meta_log_disable=0
androidboot.dtb_idx=0
androidboot.dtbo_idx=0
```

- **verifiedbootstate=orange**：Verified Boot 状态（orange 表示未验证）。
- **bootopt**：启动优化参数。
- **buildvariant**：系统类型（user/userdebug）。
- **oppo_boot_mode**：厂商启动模式。
- **bootreason**：上次启动原因。
- **atm/meta_log_disable**：厂商功能开关。
- **dtb_idx/dtbo_idx**：设备树选择。

---

### 7. 厂商定制 PHX 参数

```text
phx_rus_conf.main_on=1
phx_rus_conf.recovery_method=2
phx_rus_conf.kernel_time=240
phx_rus_conf.android_time=250
fd_passwd=43555
```

- **phx_rus_conf.main_on=1**：主流程打开。
- **recovery_method=2**：恢复方法选择。
- **kernel_time/android_time**：启动计时参数。
- **fd_passwd**：厂商调试或恢复密码。

---

### 8. 启动性能分析（Bootprof）

```text
bootprof.pl_t=2779
bootprof.lk_t=8138
bootprof.logo_t=1224
```

- **pl_t**：preloader 时间（ms）。
- **lk_t**：Little Kernel 时间（ms）。
- **logo_t**：显示 logo 时间（ms）。

---

### 9. 崩溃/转储配置（MRDump）

```text
mrdump_ddrsv=yes
mrdump_cb=0x11e000,0x2000
```

- **mrdump_ddrsv=yes**：DDR 内存保留用于崩溃转储。
- **mrdump_cb**：回调缓冲区地址和大小。

## 总结

其实也不用总结，直接背下来或者多看看就行

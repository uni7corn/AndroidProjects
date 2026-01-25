## 学习目标

1. AVB 校验 boot 是否篡改
2. dm-verity 是校验系统层面目录是否篡改
   - 阅读 AVB（Android Verified Boot）文档
   - 理解 dm-verity 工作原理
   - 标注：
     - 哪个阶段校验 Boot
     - 哪个阶段校验 System / Vendor

---

## 整体视角：Android 启动信任链（Chain of Trust）

Android 的启动安全并不是单点校验，而是一条**逐级传递的信任链**。

```
[ Boot ROM ]
      ↓
[ Bootloader ]
      ↓
[ boot.img ]
  ├─ kernel
  └─ ramdisk (init)
      ↓
[ init ]
      ↓
[ system / vendor / product 挂载 ]
```

在这条链路上，不同阶段承担不同的校验职责。

---

## 三、AVB（Android Verified Boot）概念与定位

### 3.1 AVB 是什么
**AVB 是 Android 官方定义的一套“启动完整性验证框架”**，其核心目标是：

> 确保设备从上电开始，到内核与系统启动为止，执行的每一段关键代码都未被未授权篡改。
> AVB 并不是一个单一模块，而是一整套机制的统称，包括：

- 信任根（Root of Trust）
- vbmeta 元数据结构
- 启动阶段的校验策略
- 对 dm-verity 的配置与约束

---

### 3.2 vbmeta.img 的角色

`vbmeta.img` 是 AVB 的**核心载体**，它并不包含可执行代码，而是：
> **校验清单 + 策略描述文件**

其中包含：
- boot / system / vendor 等分区的 hash 或 hashtree root
- 校验算法信息
- 校验失败时的处理策略（reboot / warning / permissive）
- 公钥与签名信息
可以将 vbmeta 理解为：
> “哪些分区必须被校验，以及如何校验”的权威声明

---

## 四、Boot 分区的校验机制（AVB 的第一道关卡）

### 4.1 Boot 分区包含什么
`boot.img` 通常包含：
- Linux kernel
- ramdisk（init 及早期用户空间）

这是系统能够继续启动的**最小执行单元**。

---

### 4.2 Boot 是在什么时候被校验的？
**结论先行：boot.img 在内核启动之前就已经被校验完成。**
### 4.3 校验发生的阶段

```
阶段：Bootloader
执行者：Bootloader（ABL / LK / U-Boot 等）
机制：AVB + vbmeta
```

### 4.4 具体校验流程

1. Bootloader 加载 vbmeta.img
2. 校验 vbmeta.img 的数字签名（信任根）
3. 从 vbmeta 中读取 boot.img 的 hash / descriptor
4. 计算当前 boot.img 的 hash
5. 对比校验结果
6. 根据策略决定：
   - 继续启动
   - 警告模式（orange state）
   - 中止启动（red state）

**关键结论：**
> boot.img 如果校验失败，内核代码根本不会获得执行机会。

---

## 五、system / vendor 的校验问题

### 5.1 一个常见误区
> “AVB 会校验 system 分区内容”

这是**不准确的**。
AVB 本身并不逐块读取 system.img 来做校验。

---

### 5.2 system / vendor 是如何被纳入 AVB 体系的
AVB 的作用是：
- 在 vbmeta 中声明：
  - system / vendor 是否启用完整性校验
  - 使用哪种校验方式
  - 校验失败如何处理
**真正执行校验的，不是 AVB，而是 dm-verity。**

---

## 六、dm-verity：运行时完整性校验机制

### 6.1 dm-verity 是什么

`dm-verity` 是 Linux Device Mapper 的一个 target，用于提供：
> **只读块设备的运行时完整性校验**

它被 Android 用于保护：
- system
- vendor
- product 等只读分区

---

### 6.2 dm-verity 校验的对象

**不是目录、不是文件，而是块设备。**
```
/dev/block/xxx → dm-verity → /system
```
校验单位是 block（通常 4KB）。

---

### 6.3 校验发生的阶段

```
阶段：Kernel 阶段（init 挂载分区之后）
触发条件：读取 block 时
```
dm-verity 并不会在开机时一次性扫描整个 system 分区，而是：
> **按需校验（lazy verification）**

---
### 6.4 dm-verity 的工作机制（简化）
1. system 分区被挂载为 verity 设备
2. 内核为该设备建立 hash tree
3. 当用户空间读取某个 block：
   - 计算该 block 的 hash
   - 向上校验至 root hash
1. 若校验失败：
   - 触发 vbmeta 中定义的策略（panic / reboot / error）

---

## 七、AVB 与 dm-verity 的职责边界总结

| 项目         | AVB            | dm-verity |
| ------------ | -------------- | --------- |
| 所在阶段     | Bootloader     | Kernel    |
| 是否执行校验 | 不直接执行     | 是        |
| 校验对象     | 各分区的元数据 | 块设备    |
| 决策权       | 有（策略）     | 无        |
| 执行权       | 无             | 有        |
一句话总结：
> **AVB 决定“校不校、怎么校”，dm-verity 负责“真正去校验”。**

---

## 八、关键阶段标注（可直接背诵）

```
[Boot 校验]
阶段：Bootloader
机制：AVB + vbmeta
对象：boot.img（kernel + ramdisk）

[System / Vendor 校验]
阶段：Kernel（init 挂载后）
机制：dm-verity
对象：system.img / vendor.img（block device）
```

---

## 九、学习检查（自测问题）

1. 为什么 system.img 被篡改后，设备仍可能成功启动？
2. dm-verity 为什么不在开机时一次性校验所有 block？
3. vbmeta 中的 root hash 被篡改会发生什么？
4. 如果关闭 AVB，但保留 dm-verity，会发生什么？
---

## 十、后续学习建议

- 阅读 AOSP 中 `fs/verity/` 与 `drivers/md/dm-verity.c`
- 实际解包并分析一台设备的 `vbmeta.img`
- 结合 init 的 mount 参数理解 verity 设备创建流程
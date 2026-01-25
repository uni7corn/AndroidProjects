# 安卓底层改机分析蓝队 · 半年系统学习路线

> **目标画像**：
> 能胜任支付宝 / 京东等大厂 **Android 底层系统改机分析 / 蓝队方向**，具备系统理解、对抗分析与检测设计能力。
>
> **学习强度**：
> - 每天 **2 小时（晚上）**
> - 每周约 **10–12 小时**
> - 半年 ≈ **240–280 小时**

---

## 总体阶段划分（6 个月）

| 阶段 | 时间 | 核心目标 |
|----|----|----|
| Phase 1 | 第 1–4 周 | Android 系统与 ART 底层补全 |
| Phase 2 | 第 5–8 周 | Native / Hook / 注入原理 |
| Phase 3 | 第 9–12 周 | Frida / Xposed / Zygisk 对抗 |
| Phase 4 | 第 13–16 周 | 改机 / 虚拟化 / 设备可信 |
| Phase 5 | 第 17–20 周 | 内核 / SELinux / 系统完整性 |
| Phase 6 | 第 21–24 周 | 蓝队检测设计 + 面试准备 |

---

## Phase 1（第 1–4 周）
### Android 系统启动 & ART 基础

### 学习目标
- 建立 **Android 不是 App，而是操作系统** 的整体认知
- 明确系统各阶段的可信边界

### 学习内容
- Android 启动流程
  - BootROM / Bootloader / Kernel / init / Zygote / SystemServer
  - AVB / dm-verity
- Android 进程模型
  - Zygote fork
  - UID / SELinux context
- ART Runtime
  - dex → oat → vdex
  - JNI 调用路径

### 每日安排（示例）
- 30 min：阅读 AOSP 源码（init / zygote）
- 30 min：博客 / 官方文档
- 60 min：源码跟读 + 画流程图

### 输出要求
- 一张 **Android 启动链路图**
- 一篇笔记：
  > “系统完整性在哪些阶段最容易被破坏？”

---

## Phase 2（第 5–8 周）
### Native 基础 & Hook 原理

### 学习目标
- 从“会用 Frida”升级为“理解 Hook 本质”

### 学习内容
- ELF 文件结构
- so 加载流程（linker）
- Hook 技术
  - inline hook（ARM64）
  - PLT / GOT hook
  - symbol hook
- ptrace / maps / auxv

### 必做实验
- 手写 inline hook（ARM64）
- 解析 `/proc/self/maps`
- 对比 Frida inline hook 实现

### 输出要求
- 一篇笔记：
  > “Android 上所有 Hook 技术的分类与优劣”

---

## Phase 3（第 9–12 周）
### Frida / Xposed / Zygisk 深度

### 学习目标
- 理解 **主流注入框架的架构差异**

### 学习内容
- Frida
  - frida-server 注入流程
  - gum-stalker trace 原理
- Xposed / LSPosed
  - Zygote 注入点
  - Method Hook 原理
- Zygisk / Riru
  - Magisk 注入链

### 对抗视角
- Frida 检测点有哪些
- Zygisk 为什么更隐蔽

### 输出要求
- 对比表：
  | 框架 | 注入时机 | 可检测点 | 蓝队对策 |

---

## Phase 4（第 13–16 周）
### 改机 / 虚拟化 / 模拟环境识别

### 学习目标
- 形成 **改机手段的完整分类体系**

### 学习内容
- 改机类型
  - build.prop
  - Framework Hook
  - Native HAL
  - Kernel / ROM
- 虚拟化
  - VMOS / 云手机
  - unidbg

### 检测维度
- 硬件一致性（sensor / cpuinfo）
- 行为一致性（timing / syscall）
- 环境一致性（binder / service）

### 输出要求
- 一篇系统文档：
  > “为什么单点检测必然失败？”

---

## Phase 5（第 17–20 周）
### Kernel / SELinux / 系统完整性

### 学习目标
- 看懂 **内核层防护与绕过**

### 学习内容
- Android Kernel 基础
- SELinux
  - enforcing / permissive
  - Magisk 绕过方式
- 完整性机制
  - dm-verity
  - AVB

### 输出要求
- 绘制：
  > “Magisk 绕过系统完整性的完整路径图”

---

## Phase 6（第 21–24 周）
### 蓝队检测设计 & 面试准备

### 学习目标
- 从工程师升级为 **防御设计者**

### 核心能力
- 组合检测设计
- 风险评分模型
- 稳定性 / 误杀控制

### 模拟面试题
- 如何设计 Frida 检测并保证可演进？
- 蓝队检测如何灰度上线？

### 最终输出
- **个人蓝队技术手册（PDF / Blog）**
- **简历项目重构（偏防御）**

---

## 推荐学习资源（精选）

- AOSP Source
- Android Internals
- Frida / Magisk / LSPosed 源码
- Linux Kernel 文档

---

## 结语

> 如果你能 **完整跑完这 6 个月**，
> 你已经不是“安卓逆向选手”，而是 **Android 系统级蓝队工程师候选人**。

后续可以：
- 拆成博客系列
- 用作大厂面试材料
- 延展到 iOS / HarmonyOS


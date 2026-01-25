# Day 9：Android UID / GID

## 学习目标
- 理解 Android UID 的设计动机
- 区分 system UID 与 app UID
- 明确 UID / GID 在权限模型中的基础作用
---

## 一、Android 为什么要设计自己的 UID 体系

Android 基于 Linux 内核，但其目标不是多用户登录系统，而是：

**每一个 App 都是一个相互隔离的独立实体。**
因此 Android 采用：
- 每个 App 分配独立 UID
- 系统服务使用固定 system UID
- 权限校验主要发生在 Binder + system_server 层
UID 是 Android 安全模型中的**第一层隔离机制**。

---

## 二、Android UID 的整体划分

Android 的 UID 数值是有明确区间含义的。

| UID 区间      | 含义               |
| ----------- | ---------------- |
| 0           | root             |
| 1–9999      | system UID（系统进程） |
| 10000–19999 | app UID（普通应用）    |
| ≥ 20000     | 多用户 / 隔离 UID     |

相关定义位于源码： `system/core/libcutils/include/private/android_filesystem_config.h`

---

## 三、system UID（系统 UID）

### 1. 什么是 system UID
system UID 用于 Android 系统级服务和守护进程，例如：

| UID  | 名称        | 典型进程          |
| ---- | --------- | ------------- |
| 0    | root      | init          |
| 1000 | system    | system_server |
| 1001 | radio     | rild          |
| 1002 | bluetooth | bluetoothd    |
| 1013 | media     | mediaserver   |

### 2. system UID 的特点
- system UID **不等同于 root**
- 仍然受文件权限与 SELinux 限制
- 系统能力主要来源于 Binder 与 SELinux policy

示例：
system_server  
UID = 1000  
SELinux domain = system_server

---

## 四、app UID（应用 UID）

### 1. App UID 的分配规则

普通 App 的 UID 从 10000 开始：
UID = AID_APP_START + appId

查看方式：
dumpsys package <package_name> | grep userId

### 2. app UID 的作用
- 文件隔离（/data/data/<package>）
- 进程隔离
- 权限隔离
不同 UID 的 App 默认无法互相访问数据

---

## 五、权限与 UID 的关系（重点）

**UID 本身不是权限。**
真实权限判定流程为：
1. App 通过 Binder 调用系统服务
2. Binder 传递调用方 UID
3. system_server 校验：
    - UID 对应的包名
    - 声明的权限
    - 授权状态

结论：
权限 ≠ UID  
UID = 身份标识

---

## 六、GID 在 Android 中的作用

GID 用于补充权限能力，常见用途：
- 网络访问（inet）
- 媒体读写（media_rw）

示例：
- 申请 INTERNET 权限 → 进程加入 inet GID
可以理解为：
- UID：我是谁
- GID：我额外属于哪些能力组

---

## 七、总结
- Android 以 UID 作为基础安全隔离单元
- system UID 与 app UID 有明确职责边界
- UID 决定身份，不直接决定权限
- 真实权限由 Binder + system_server + SELinux 决定
# Day 12：Binder IPC 学习笔记

AI 直接生产的，我自己并不懂，我想等到学习 LSP 时候再细节去学

> 目标定位：  
> 本笔记用于**建立 Binder 的系统级认知**，而非 App 开发或 AIDL 使用指南。  
> 重点回答三个问题：
>
> 1. Binder 为什么存在
> 2. 一次 Binder IPC 真实发生了什么
> 3. 为什么必须存在客户端 / 服务端（Bp / Bn）

---

## 一、Binder 的存在意义

### 1. Android 的核心约束

Android 是一个：

- 多进程
- 强 UID 隔离
- 强 SELinux 隔离
- 系统服务高度集中（system_server）

系统必须解决的问题是：

> **普通 App 如何在严格安全模型下，调用高权限系统服务？**

传统 IPC（socket / pipe / shm）的问题：

- 无法可信地标识调用者身份
- 权限校验难以统一
- 不适合“服务化调用模型”

### 2. Binder 的一句话定义

> **Binder 是一个内核级、携带身份信息的、基于对象引用的 RPC 机制**

它解决的是：

- 身份可信（UID / PID / SELinux context）
- 调用可控
- 服务集中化管理

---

## 二、Binder 的整体架构

### 1. 三层结构

```
[ App Process ]
      │
      │  Binder 调用（transact）
      ▼
===========================
|   Binder Driver (Kernel) |
| - 校验 UID / PID         |
| - handle → node 查表    |
| - 线程调度 / 唤醒       |
===========================
      │
      ▼
[ system_server ]
  Binder Thread Pool
      │
      ▼
[ System Service ]
```

Binder Driver 是 **唯一可信仲裁者**。

---

## 三、一次 Binder IPC 的完整调用流程

### 示例：App 调用 ActivityManagerService

#### Step 1：获取 Binder 对象

```java
IBinder binder = ServiceManager.getService("activity");
```

含义：

- 向 ServiceManager 请求名为 `activity` 的 Binder 服务
- 返回的是 **IBinder 代理对象**

---

#### Step 2：转换为接口代理（Bp）

```java
IActivityManager am = IActivityManager.Stub.asInterface(binder);
```

此时：

- `am` 并不是 AMS
- 而是 `BpActivityManager`

---

#### Step 3：发起调用

```java
am.getConfiguration();
```

Bp 内部行为：

- 创建 Parcel
- 写入参数
- 调用 `transact()`

---

#### Step 4：进入内核 Binder 驱动

- `ioctl(BINDER_WRITE_READ)`
- 内核记录 calling UID / PID
- 根据 handle 找到目标进程
- 将事务放入 system_server 队列
- 唤醒 Binder 线程

---

#### Step 5：system_server 处理事务

```
Binder Thread
   ↓
BnActivityManager.onTransact()
   ↓
ActivityManagerService.getConfiguration()
```

权限校验发生在 **服务端逻辑中**。

---

#### Step 6：返回结果

- 服务端写回 Parcel
- Binder Driver 负责拷贝与唤醒
- 客户端线程恢复执行

---

## 四、Binder 的核心抽象

### 1. 四个关键角色

| 角色            | 作用             |
| ------------- | -------------- |
| IBinder       | 跨进程对象抽象        |
| BpXXX         | 客户端代理（协议编码）    |
| BnXXX         | 服务端 Stub（协议解码） |
| Binder Driver | 内核仲裁与调度        |

一句话：

> **Bp 是“假对象”，Bn 是“真对象”，内核负责连接二者**

---

### 2. Handle 与对象引用

- Binder 不传递内存指针
- 只传递 `handle`（整数）

内核维护映射：

```
handle → binder_node → target process
```

安全性保证：

- handle 不可伪造
- 引用生命周期由内核管理

---

## 五、为什么必须存在客户端 / 服务端

### 1. Binder 不理解业务语义

内核只理解：

- buffer
- handle
- transaction code

内核**不能理解**：

- startActivity
- installPackage

否则等于把 Android Framework 写进内核。

---

### 2. Binder 是通用 IPC 机制

Binder 同时服务于：

- App ↔ System Server
- App ↔ App
- HAL ↔ Framework
- Vendor ↔ System

因此：

> **RPC 语义必须由用户态定义**

---

### 3. 权限校验发生在服务端

```java
Binder.getCallingUid();
```

内核只负责告诉服务端：

- 谁在调用

是否允许：

- 由服务端逻辑决定

因此服务端必须：

- 知道调用的是哪个方法
- 理解参数含义

---

## 六、Bp / Bn / onTransact 的真实职责

| 组件         | 本质职责                |
| ---------- | ------------------- |
| BpXXX      | 协议编码器（write Parcel） |
| BnXXX      | 协议解码器（read Parcel）  |
| onTransact | 请求分发器               |

这些并非 Binder 多余设计，而是 **RPC 不可删除的结构**。

---

## 七、今日学习总结

- Binder 解决的是 **安全可控的跨进程服务调用**
- Binder 只负责通道，不负责业务语义
- 客户端 / 服务端是 RPC 的必然结构
- Android 的权限模型建立在 Binder 之上

一句话总结：

> **Binder 是 Android 的“权力总线”，而不是简单的 IPC 工具。**

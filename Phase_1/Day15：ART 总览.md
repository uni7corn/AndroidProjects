# Day 15：ART 总览（Android 12 视角）

> 目标：**理解 ART 是什么、干什么、以及 Dalvik 与 ART 的本质差异**。
> 本文可直接作为学习笔记或复习提纲。

---

## 一、ART 是什么（一句话）

**ART（Android Runtime）是 Android 的应用运行时系统**，负责执行 DEX 字节码，并提供：

- 代码执行（解释 / JIT / AOT）
- 内存管理（GC）
- 线程与异常处理
- JNI 桥接
- ClassLoader / 反射支持

> 可类比为：**Android 世界里的 JVM + 执行引擎 + 内存管理器**。

---

## 二、ART 的核心职责

### 1. 执行 DEX（最核心）

编译链路：

```
Java / Kotlin
   ↓
.class
   ↓ (d8)
.dex
   ↓
ART 执行
```

ART 支持三种执行方式：

- **Interpreter**：解释执行（冷启动、调试）
- **JIT**：运行时即时编译（热路径）
- **AOT**：安装/空闲时提前编译（dex2oat）

---

### 2. 内存管理（GC）

- 分代 GC
- 并发 / 并行 GC
- Card Table / Write Barrier
- TLAB（线程本地分配缓冲）

> 应用卡顿、OOM 与 ART GC 强相关。

---

### 3. ClassLoader 与反射

- PathClassLoader
- DexClassLoader
- 动态加载 dex / so
- 反射调用 Method.invoke

> 插件化、热修复、Hook 的基础。

---

### 4. JNI 桥接

- Java ↔ Native（C/C++）
- 管理 JNI 引用（Local / Global）
- 线程 attach / detach

> **JNI 从 Dalvik 时代就存在**，不是 ART 新增。

---

### 5. 线程、异常、调试

- Java Thread → Linux Thread
- Java 异常展开
- JDWP 调试支持

---

## 三、Dalvik vs ART（核心对比）

| 对比项       | Dalvik   | ART                     |
| ------------ | -------- | ----------------------- |
| Android 版本 | ≤ 4.4    | ≥ 5.0                   |
| 执行模型     | JIT 为主 | Interpreter + JIT + AOT |
| 编译时机     | 运行时   | 安装时 / 运行时         |
| 启动性能     | 较慢     | 更快                    |
| 运行稳定性   | 抖动较多 | 更稳定                  |

**一句话总结：**

- Dalvik：边跑边编
- ART：能提前编的尽量提前编

---

## 四、为什么从 Dalvik 迁移到 ART

Dalvik 的问题：

- 冷启动慢
- JIT 抖动明显
- GC 停顿大

ART 的改进：

- AOT 提升启动速度
- JIT + Profile 精准优化
- 更先进的 GC
- 更省电、更稳定

---

## 五、重要澄清：ART ≠ 纯 AOT

现代 ART（Android 7+）执行模型：

```
冷代码 → 解释执行
热代码 → JIT
稳定后 → Profile-guided AOT
```

优点：

- 安装更快
- OTA 可重优化
- 系统更灵活

---

## 六、Android 12 的 ART 特点

### 1. 完全取代 Dalvik

- Android 12 **100% 使用 ART**
- Dalvik VM 已不存在

### 2. ART 模块化（APEX）

路径示例：

```
/apex/com.android.art/
```

> ART 可独立于系统升级。

### 3. 名称遗留不等于 Dalvik 存在

仍可见的历史命名：

- .dex
- dalvik-cache
- dalvik.vm.\* 属性

这些只是**历史遗留命名**。

---

## 七、从安全 / 逆向视角看 ART

- Java Hook 难度 ↑（inline / deopt / JIT cache）
- Native Hook 价值 ↑（JNI / libc / linker）
- Framework 越来越多逻辑下沉到 Native

> **真正的 Android 高级逆向 = ART 层理解**。

---

## 八、Day 15 达成标准（不要过度学习）

- [ ] 知道 ART 是什么
- [ ] 能区分 Dalvik vs ART
- [ ] 明白 ART 不是纯 AOT
- [ ] 不需要看源码
- [ ] 不需要深挖 GC

**到此为止即可**
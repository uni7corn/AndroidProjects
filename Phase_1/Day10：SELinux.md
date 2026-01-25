## 学习目标

1. 理解 Android SELinux 模式（Enforcing）
2. 阅读 sepolicy 结构
3. 使用命令查看：
   - `ps -Z`
   - `ls -Z`

---

## SELinux

SELinux（Security-Enhanced Linux）是一种 Linux 内核安全模块，用于强制访问控制（MAC, Mandatory Access Control）。Android 中 SELinux 的引入，是为了限制应用和系统组件对敏感资源的访问，提高系统安全性

| 模式             | 含义                           |
| -------------- | ---------------------------- |
| **Enforcing**  | 强制模式，SELinux 策略严格生效，非法访问被阻止。 |
| **Permissive** | 宽容模式，SELinux 仅记录违规访问，不阻止。    |
| **Disabled**   | SELinux 被关闭，系统不进行访问控制。       |

在你不改的情况下，基本都是强制模式，你获取 root 以后可以改成宽容模式，级别没人会完全关闭，可以通过`getenforce`来查看
接下来看`te`文件，就是 selinux 的配置文件

```shell
##
# trusted execution environment (tee) daemon 这里说明是TEE的策略，这种注释在项目里面很常见
#
type tee, domain;

# Device(s) for communicating with the TEE
type tee_device, dev_type;

allow tee fingerprint_vendor_data_file:dir rw_dir_perms;
allow tee fingerprint_vendor_data_file:file create_file_perms;
```

- `type tee, domain;`
  - 定义一个新的 **域**（domain）`tee`。
  - 域就是一个进程运行的安全上下文类型，例如 `zygote`、`untrusted_app`。
  - **domain** 表示它是进程类型，可以被分配给进程。
- `type tee_device, dev_type;`
  - 定义一个新的 **对象类型**（type）`tee_device`。
  - **dev_type** 表示这是一个设备类型，用于 `/dev/...` 设备文件。

接下来就是规则部分`allow`，先看语法

```shell
allow <source_domain> <target_type>:<class> <permissions>;
```

1. 第一行里的内容
   - `source_domain` = `tee`（TEE 守护进程）
   - `target_type` = `fingerprint_vendor_data_file`（指纹厂商数据目录）
   - `class` = `dir`（目录类型）
   - `permissions` = `rw_dir_perms`（预定义的目录读写权限集合，包括 `read`、`write`、`search` 等）
2. 第二行里面的内容 - `source_domain` = `tee`（TEE 守护进程） - `target_type` = `fingerprint_vendor_data_file`（指纹厂商数据文件） - `class` = `file`（普通文件） - `permissions` = `create_file_perms`（预定义文件权限集合，包括 `read`、`write`、`append`、`create` 等）
   te`文件在 Android 里面并不是这种直接读的，而是编译成二进制

---

我们来看个案例

```text
LABEL                          USER           PID  PPID     VSZ    RSS WCHAN            ADDR S NAME
u:r:su:s0                      root         22209 22201 10840372  3384 SyS_rt_si+          0 S sh
u:r:su:s0                      root         22530 22209 10838876  3180 0                   0 R ps
```

`LABEL`部分的格式是`user:role:type:level`

- **user** → `u`
  - SELinux 用户，这里是 `u`，Android 常用默认用户。
- **role** → `r`
  - SELinux 角色，这里是 `su` 进程对应的角色。
- **type/domain** → `su`
  - 这是最重要的，决定进程属于哪个 **域（domain）**，即它能访问哪些对象。
  - 这里 `su` 进程属于 `su` 域。
- **level** → `s0`
  - MCS/MLS 标签，多数 Android 上的多类别安全（MCS）用于隔离应用。
  - 对系统服务通常是 `s0`。

---

## 总结

我现在感觉总是莫名其妙就总结了，但是内容也讲完了，这个比较抽象，各位多看看

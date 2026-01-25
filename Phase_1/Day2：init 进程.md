## 学习目标
1. 下载 / 浏览 AOSP 源码
2. 定位 `system/core/init/main.cpp`
3. 理解 init 是如何成为 **PID 1** 的
4. 阅读并理解：
    - init.rc
    - service / action 语义
---
下载源码这一块，我选择下载的是LineageOS的源码，安卓12版本
是我在编译小米8的时候下载的，正好现在可以用来学习安卓系统，其实不用专门这样做，找一个网页上的源码就行了
# main.cpp
这里面就是init进程的主函数入口了
``` cpp
int main(int argc, char **argv)
{
#if __has_feature(address_sanitizer)
	__asan_set_error_report_callback(AsanReportCallback);
#endif
	// Boost prio which will be restored later
	/*

	    | 常量           | 含义   |
	    | ------------ | ---- |
	    | PRIO_PROCESS | 按进程  |
	    | PRIO_PGRP    | 按进程组 |
	    | PRIO_USER    | 按用户  | 

	PRIO_PROCESS下0指当前进程，一般开发写gitpid但是我们这里绝对意义上是当前进程1，语义上也可以直接写0


	-20  ← 最高优先级
	  0  ← 默认
	+19  ← 最低优先级
	这个就是一个范围了，我们第一个进程肯定是最高级别，所以直接写-20
	*/
	setpriority(PRIO_PROCESS, 0, -20);
	if (!strcmp(basename(argv[0]), "ueventd")) {
		return ueventd_main(argc, argv);
	}

	if (argc > 1) {
		if (!strcmp(argv[1], "subcontext")) {
			android::base::InitLogging(argv, &android::base::KernelLogger);
			const BuiltinFunctionMap &function_map = GetBuiltinFunctionMap();

			return SubcontextMain(argc, argv, &function_map);
		}

		if (!strcmp(argv[1], "selinux_setup")) {
			return SetupSelinux(argv);
		}

		if (!strcmp(argv[1], "second_stage")) {
			return SecondStageMain(argc, argv);
		}
	}

	return FirstStageMain(argc, argv);
}
```
代码很简单很简单，重要的是启动的顺序，调用的逻辑是什么
## 启动顺序
1. 第一次启动的阶段调用的是`FirstStageMain`这个函数，来进行一些初始化
2. 第二次启动阶段调用的是`SetupSelinux`这个函数，来进行`selinux`的启动加载
3. 第三次启动阶段调用的是`SecondStageMain`这个函数，来进行第二次的启动
	这个启动，里面会加载rc文件，会进入Zygote然后就进入到安卓的世界了
具体的内容，计划后面都会有东西的，不必太在意

## pid 1
`init` 进程由内核直接启动，是用户空间的第一个进程（PID 1），负责：
- 挂载文件系统
- 初始化系统属性
- 启动和管理系统服务
- 处理孤儿进程回收
## init.rc
这玩意一千多行，拿不上来
先学service / action是什么意思，然后再去看，配合init的main函数去看
会发现很多很多内容的，有的启动是怎么启动的，我是没看全，不过以后肯定是会全部看下来的
前期不要给自己那么大的压力，以免自己不想学了
### service 和 action
#### service 启动进程
``` shell
service boringssl_self_test64 /system/bin/boringssl_self_test64
    setenv BORINGSSL_SELF_TEST_CREATE_FLAG true # Any nonempty value counts as true
    reboot_on_failure reboot,boringssl-self-check-failed
    stdio_to_kmsg

service boringssl_self_test_apex64 /apex/com.android.conscrypt/bin/boringssl_self_test64
    setenv BORINGSSL_SELF_TEST_CREATE_FLAG true # Any nonempty value counts as true
    reboot_on_failure reboot,boringssl-self-check-failed
    stdio_to_kmsg
```
它的格式是
``` shell
service <name> <executable> [args...]
    [class <class>]
    [user <user>]
    [group <group>]
    [oneshot]         # 执行一次就退出
    [disabled]        # 默认不启动
```
含义就是：
- `service` 定义了 **一个守护进程（daemon）**，由 init 启动并管理生命周期。
- 它可以指定：
    - 所属 **class**（方便按类别启动/停止，如 core、main、late_start）
    - 运行的 **用户/组**
    - 是否是 **oneshot**（一次性执行，不会重启）
    - 是否 **disabled**（默认不启动，需要 `start <name>` 手动启动）
#### action 响应事件
这个好理解很多了，没那么多的参数
``` shell
on early-init && property:ro.product.cpu.abilist32=*
    exec_start boringssl_self_test32
on early-init && property:ro.product.cpu.abilist64=*
    exec_start boringssl_self_test64
on property:apexd.status=ready && property:ro.product.cpu.abilist32=*
    exec_start boringssl_self_test_apex32
on property:apexd.status=ready && property:ro.product.cpu.abilist64=*
    exec_start boringssl_self_test_apex64
```
这个的格式是
``` shell
on <event> [mask <mask>]
    <command1>
    <command2>
```

**含义**：
- `action` 是 **事件触发机制**。它告诉 init：**当某个事件发生时，执行哪些命令**。
- 常用事件：
    - `boot` → 系统启动时
    - `property:<key>=<value>` → 某个系统属性改变
    - `fs` → 文件系统挂载完成
    - `uevent` → 内核设备事件
- `action` 里的命令可以是：
    - `start <service>` → 启动服务
    - `setprop <key> <value>` → 设置系统属性
    - 其他 shell 命令
这两个是可以组合使用的，service不是立刻启动，它是一个定义好的服务，action是事件驱动型的，发生它标记的事件以后就会执行，action可以执行service内的任务
``` shell
service ueventd /system/bin/ueventd # 这里是一个定义
    class core
    critical
    seclabel u:r:ueventd:s0
    shutdown critical
on early-init # 这个系统前期初始化做的太多了，就拿一个案例出来
    start ueventd
```
上面代码段就是一个小案例

## 结束
以上就是简单的启动，我感觉现在东西还是很浅显的，前期任务我是别推荐太大，免得自己完不成
简单看看逻辑得了，后面再深入学习
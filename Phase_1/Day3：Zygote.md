## 学习目标
1. 定位 Zygote 启动入口
2. 理解 Zygote 与普通 Linux 进程的区别
3. 理解：
  - 为什么 Android 要用 Zygote
  - fork() 在这里解决了什么问题
---
## Zygote入口
在第二次init启动里加载的rc文件里面[[Day2：init 进程]]，具体的文件内容是
``` shell
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks
    critical window=${zygote.critical_window.minute:-off} target=zygote-fatal
```
我拿的是64位的这个，这个服务启动后，我们就进入了Zygote的世界里面 `app_process`地址在`frameworks/base/cmds/app_process/app_main.cpp`，具体函数就不贴了，就贴一下入口吧
``` cpp
	if (zygote) {
		runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
	} else if (className) {
		runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
	} else {
		fprintf(stderr, "Error: no class name or --zygote supplied.\n");
		app_usage();
		LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
	}
```
启动后就进入Java的世界了，到此我们找的是Zygote的入口
## Zygote进程和fork进程有什么区别
区别其实不大，只不过是Zygote进程多了很多Java和安卓framework的依赖库，因为fork的系统机制，所以Zygote的作用就相当于预加载一些基本库，fork是不会加载的，为了提高性能，所以安卓会从Zygote进行后续安卓方面进程的加载和使用

## 结语
在写完之后我才发现，这一篇好像已经写完了，提到了新内容，但是内容本身不多也不难
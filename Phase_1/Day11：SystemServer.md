## 学习目标

- 理解 SystemServer 的特殊地位
- 阅读 SystemServer 启动流程
- 记录核心服务启动顺序

---

## 特殊地位

---

## 启动流程

这个的启动函数还是在[[Day3：Zygote]]中的`main`函数中，叫作`forkSystemServer`

```Java
if (startSystemServer) {
	Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

	// {@code r == null} in the parent (zygote) process, and {@code r != null} in the
	// child (system_server) process.
	if (r != null) {
		r.run();
		return;
	}
}
```
if判断的是是否启动系统服务，里面的`forkSystemServer`就是我们今天追的函数了，也不是很难的
``` java
private static Runnable forkSystemServer(String abiList, String socketName, ZygoteServer zygoteServer)
{
	long capabilities = posixCapabilitiesAsBits(
		OsConstants.CAP_IPC_LOCK,
		OsConstants.CAP_KILL,
		OsConstants.CAP_NET_ADMIN,
		OsConstants.CAP_NET_BIND_SERVICE,
		OsConstants.CAP_NET_BROADCAST,
		OsConstants.CAP_NET_RAW,
		OsConstants.CAP_SYS_MODULE,
		OsConstants.CAP_SYS_NICE,
		OsConstants.CAP_SYS_PTRACE,
		OsConstants.CAP_SYS_TIME,
		OsConstants.CAP_SYS_TTY_CONFIG,
		OsConstants.CAP_WAKE_ALARM,
		OsConstants.CAP_BLOCK_SUSPEND);
	/* Containers run without some capabilities, so drop any caps that are not available. */
	StructCapUserHeader header = new StructCapUserHeader(
		OsConstants._LINUX_CAPABILITY_VERSION_3,
		0);
	StructCapUserData[] data;
	try {
		data = Os.capget(header);
	} catch (ErrnoException ex) {
		throw new RuntimeException("Failed to capget()", ex);
	}
	capabilities &= ((long)data[0].effective) | (((long)data[1].effective) << 32);
                                                                                                                 
	/* Hardcoded command line to start the system server */
	String[] args = {
		"--setuid=1000",
		"--setgid=1000",
		"--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,"
			+ "1024,1032,1065,3001,3002,3003,3006,3007,3009,3010,3011,3012",
		"--capabilities=" + capabilities + "," + capabilities,
		"--nice-name=system_server",
		"--runtime-args",
		"--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
		"com.android.server.SystemServer",
	};
	ZygoteArguments parsedArgs;
                                                                                                                 
	int pid;
                                                                                                                 
	try {
		ZygoteCommandBuffer commandBuffer = new ZygoteCommandBuffer(args);
		try {
			parsedArgs = ZygoteArguments.getInstance(commandBuffer);
		} catch (EOFException e) {
			throw new AssertionError("Unexpected argument error for forking system server", e);
		}
		commandBuffer.close();
		Zygote.applyDebuggerSystemProperty(parsedArgs);
		Zygote.applyInvokeWithSystemProperty(parsedArgs);
                                                                                                                 
		if (Zygote.nativeSupportsMemoryTagging()) {
			/* The system server has ASYNC MTE by default, in order to allow
         * system services to specify their own MTE level later, as you
         * can't re-enable MTE once it's disabled. */
			String mode = SystemProperties.get("arm64.memtag.process.system_server", "async");
			if (mode.equals("async")) {
				parsedArgs.mRuntimeFlags |= Zygote.MEMORY_TAG_LEVEL_ASYNC;
			} else if (mode.equals("sync")) {
				parsedArgs.mRuntimeFlags |= Zygote.MEMORY_TAG_LEVEL_SYNC;
			} else if (!mode.equals("off")) {
				/* When we have an invalid memory tag level, keep the current level. */
				parsedArgs.mRuntimeFlags |= Zygote.nativeCurrentTaggingLevel();
				Slog.e(TAG, "Unknown memory tag level for the system server: \"" + mode + "\"");
			}
		} else if (Zygote.nativeSupportsTaggedPointers()) {
			/* Enable pointer tagging in the system server. Hardware support for this is present
         * in all ARMv8 CPUs. */
			parsedArgs.mRuntimeFlags |= Zygote.MEMORY_TAG_LEVEL_TBI;
		}
                                                                                                                 
		/* Enable gwp-asan on the system server with a small probability. This is the same
     * policy as applied to native processes and system apps. */
		parsedArgs.mRuntimeFlags |= Zygote.GWP_ASAN_LEVEL_LOTTERY;
                                                                                                                 
		if (shouldProfileSystemServer()) {
			parsedArgs.mRuntimeFlags |= Zygote.PROFILE_SYSTEM_SERVER;
		}
                                                                                                                 
		/* Request to fork the system server process */
		pid = Zygote.forkSystemServer(
			parsedArgs.mUid,
			parsedArgs.mGid,
			parsedArgs.mGids,
			parsedArgs.mRuntimeFlags,
			null,
			parsedArgs.mPermittedCapabilities,
			parsedArgs.mEffectiveCapabilities);
	} catch (IllegalArgumentException ex) {
		throw new RuntimeException(ex);
	}
                                                                                                                 
	/* For child process */
	if (pid == 0) {
		if (hasSecondZygote(abiList)) {
			waitForSecondaryZygote(socketName);
		}
                                                                                                                 
		zygoteServer.closeServerSocket();
		return handleSystemServerProcess(parsedArgs);
	}
                                                                                                                 
	return null;
}
```

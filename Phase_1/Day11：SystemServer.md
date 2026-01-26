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

这个函数调用`handleSystemServerProcess`函数，然后走了`zygoteInit`函数

``` java
private static Runnable handleSystemServerProcess(ZygoteArguments parsedArgs) {
    // set umask to 0077 so new files and directories will default to owner-only permissions.
    Os.umask(S_IRWXG | S_IRWXO);
                                                                                               
    if (parsedArgs.mNiceName != null) {
        Process.setArgV0(parsedArgs.mNiceName);
    }
                                                                                               
    final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
    if (systemServerClasspath != null) {
        // Capturing profiles is only supported for debug or eng builds since selinux normally
        // prevents it.
        if (shouldProfileSystemServer() && (Build.IS_USERDEBUG || Build.IS_ENG)) {
            try {
                Log.d(TAG, "Preparing system server profile");
                final String standaloneSystemServerJars =
                        Os.getenv("STANDALONE_SYSTEMSERVER_JARS");
                final String systemServerPaths = standaloneSystemServerJars != null
                        ? String.join(":", systemServerClasspath, standaloneSystemServerJars)
                        : systemServerClasspath;
                prepareSystemServerProfile(systemServerPaths);
            } catch (Exception e) {
                Log.wtf(TAG, "Failed to set up system server profile", e);
            }
        }
    }
                                                                                               
    if (parsedArgs.mInvokeWith != null) {
        String[] args = parsedArgs.mRemainingArgs;
        // If we have a non-null system server class path, we'll have to duplicate the
        // existing arguments and append the classpath to it. ART will handle the classpath
        // correctly when we exec a new process.
        if (systemServerClasspath != null) {
            String[] amendedArgs = new String[args.length + 2];
            amendedArgs[0] = "-cp";
            amendedArgs[1] = systemServerClasspath;
            System.arraycopy(args, 0, amendedArgs, 2, args.length);
            args = amendedArgs;
        }
                                                                                               
        WrapperInit.execApplication(parsedArgs.mInvokeWith,
                parsedArgs.mNiceName, parsedArgs.mTargetSdkVersion,
                VMRuntime.getCurrentInstructionSet(), null, args);
                                                                                               
        throw new IllegalStateException("Unexpected return from WrapperInit.execApplication");
    } else {
        ClassLoader cl = getOrCreateSystemServerClassLoader();
        if (cl != null) {
            Thread.currentThread().setContextClassLoader(cl);
        }
                                                                                               
        /*
         * Pass the remaining arguments to SystemServer.
         */
        return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                parsedArgs.mDisabledCompatChanges,
                parsedArgs.mRemainingArgs, cl);
    }
                                                                                               
    /* should never reach here */
}

```

然后`ZygoteInit`内部是

``` java
/**
 * The main function called when started through the zygote process. This could be unified with
 * main(), if the native code in nativeFinishInit() were rationalized with Zygote startup.<p>
 *
 * Current recognized args:
 * <ul>
 * <li> <code> [--] &lt;start class name&gt;  &lt;args&gt;
 * </ul>
 *
 * @param targetSdkVersion target SDK version
 * @param disabledCompatChanges set of disabled compat changes for the process (all others
 *                              are enabled)
 * @param argv             arg strings
 */
public static Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
		String[] argv, ClassLoader classLoader) {
	if (RuntimeInit.DEBUG) {
		Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
	}

	Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
	RuntimeInit.redirectLogStreams();

	RuntimeInit.commonInit();
	ZygoteInit.nativeZygoteInit();
	return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
			classLoader);
}
```

从这里我们可以看出，系统服务只是一个比较特殊的app，然后我们来看应用初始化函数

``` java
protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
        String[] argv, ClassLoader classLoader) {
    // If the application calls System.exit(), terminate the process
    // immediately without running any shutdown hooks.  It is not possible to
    // shutdown an Android application gracefully.  Among other things, the
    // Android runtime shutdown hooks close the Binder driver, which can cause
    // leftover running threads to crash before the process actually exits.
    nativeSetExitWithoutCleanup(true);
                                                                                              
    VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
    VMRuntime.getRuntime().setDisabledCompatChanges(disabledCompatChanges);
                                                                                              
    final Arguments args = new Arguments(argv);
                                                                                              
    // The end of of the RuntimeInit event (see #zygoteInit).
    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                                                                                              
    // Remaining arguments are passed to the start class's static main
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}
```

然后走到了我们[[Day8：Zygote fork 机制]]里面最后阶段的静态选择main方法里面，找到main方法然后开始执行

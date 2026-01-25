## 学习目标
- [ ] 跟读 Zygote fork 源码
- [ ] 理解 fork 后：
  - [ ] UID 如何分配
  - [ ] 权限如何隔离

---

### Zygote 进程 fork
从[[Day5：启动参数]]这可以知道启动是在什么地方启动的，今天这就来跟进入 Java 以后是什么样子的，这个文件的路径是 `frameworks/base/core/java/com/android/internal/os`

```Java
	public static void main(String[] argv)
	{
		ZygoteServer zygoteServer = null;

		// Mark zygote start. This ensures that thread creation will throw
		// an error.
		ZygoteHooks.startZygoteNoThreadCreation();

		// Zygote goes into its own process group.
		try {
			Os.setpgid(0, 0);
		} catch (ErrnoException ex) {
			throw new RuntimeException("Failed to setpgid(0,0)", ex);
		}

		Runnable caller;
		try {
			// Store now for StatsLogging later.
			final long startTime = SystemClock.elapsedRealtime();
			final boolean isRuntimeRestarted = "1".equals(SystemProperties.get("sys.boot_completed"));

			String bootTimeTag = Process.is64Bit() ? "Zygote64Timing" : "Zygote32Timing";
			TimingsTraceLog bootTimingsTraceLog = new TimingsTraceLog(bootTimeTag, Trace.TRACE_TAG_DALVIK);
			bootTimingsTraceLog.traceBegin("ZygoteInit");
			RuntimeInit.preForkInit();

			boolean startSystemServer = false;
			String zygoteSocketName = "zygote";
			String abiList = null;
			boolean enableLazyPreload = false;
			for (int i = 1; i < argv.length; i++) {
				if ("start-system-server".equals(argv[i])) {
					startSystemServer = true;
				} else if ("--enable-lazy-preload".equals(argv[i])) {
					enableLazyPreload = true;
				} else if (argv[i].startsWith(ABI_LIST_ARG)) {
					abiList = argv[i].substring(ABI_LIST_ARG.length());
				} else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
					zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
				} else {
					throw new RuntimeException("Unknown command line argument: " + argv[i]);
				}
			}

			final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);
			if (!isRuntimeRestarted) {
				if (isPrimaryZygote) {
					FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED,
						BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__ZYGOTE_INIT_START,
						startTime);
				} else if (zygoteSocketName.equals(Zygote.SECONDARY_SOCKET_NAME)) {
					FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED,
						BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__SECONDARY_ZYGOTE_INIT_START,
						startTime);
				}
			}

			if (abiList == null) {
				throw new RuntimeException("No ABI list supplied.");
			}

			// In some configurations, we avoid preloading resources and classes eagerly.
			// In such cases, we will preload things prior to our first fork.
			if (!enableLazyPreload) {
				bootTimingsTraceLog.traceBegin("ZygotePreload");
				EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
					SystemClock.uptimeMillis());
				preload(bootTimingsTraceLog);
				EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
					SystemClock.uptimeMillis());
				bootTimingsTraceLog.traceEnd(); // ZygotePreload
			}

			// Do an initial gc to clean up after startup
			bootTimingsTraceLog.traceBegin("PostZygoteInitGC");
			gcAndFinalize();
			bootTimingsTraceLog.traceEnd(); // PostZygoteInitGC

			bootTimingsTraceLog.traceEnd(); // ZygoteInit

			Zygote.initNativeState(isPrimaryZygote);

			ZygoteHooks.stopZygoteNoThreadCreation();

			zygoteServer = new ZygoteServer(isPrimaryZygote);

			if (startSystemServer) {
				Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

				// {@code r == null} in the parent (zygote) process, and {@code r != null} in the
				// child (system_server) process.
				if (r != null) {
					r.run();
					return;
				}
			}

			Log.i(TAG, "Accepting command socket connections");

			// The select loop returns early in the child process after a fork and
			// loops forever in the zygote.
			caller = zygoteServer.runSelectLoop(abiList);
		} catch (Throwable ex) {
			Log.e(TAG, "System zygote died with fatal exception", ex);
			throw ex;
		} finally {
			if (zygoteServer != null) {
				zygoteServer.closeServerSocket();
			}
		}

		// We're in the child process and have exited the select loop. Proceed to execute the
		// command.
		if (caller != null) {
			caller.run();
		}
	}
```

虽然代码量总体不大，但是这里不会完全看，今天只看 fork 机制的使用就好了
`runSelectLoop`这个函数，是安卓一切准备好后进入监听模式，然后开始监听系统任务，创建服务或者 app
这个函数在`ZygoteServer.java`里面，接下来我们来看这个函数

``` java
	Runnable runSelectLoop(String abiList)
	{
		ArrayList<FileDescriptor> socketFDs = new ArrayList<>();
		ArrayList<ZygoteConnection> peers = new ArrayList<>();

		socketFDs.add(mZygoteSocket.getFileDescriptor());
		peers.add(null);

		mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;

		while (true) {
			fetchUsapPoolPolicyPropsWithMinInterval();
			mUsapPoolRefillAction = UsapPoolRefillAction.NONE;

			int[] usapPipeFDs = null;
			StructPollfd[] pollFDs;

			// Allocate enough space for the poll structs, taking into account
			// the state of the USAP pool for this Zygote (could be a
			// regular Zygote, a WebView Zygote, or an AppZygote).
			if (mUsapPoolEnabled) {
				usapPipeFDs = Zygote.getUsapPipeFDs();
				pollFDs = new StructPollfd[socketFDs.size() + 1 + usapPipeFDs.length];
			} else {
				pollFDs = new StructPollfd[socketFDs.size()];
			}

			/*
             * For reasons of correctness the USAP pool pipe and event FDs
             * must be processed before the session and server sockets.  This
             * is to ensure that the USAP pool accounting information is
             * accurate when handling other requests like API deny list
             * exemptions.
             */

			int pollIndex = 0;
			for (FileDescriptor socketFD : socketFDs) {
				pollFDs[pollIndex] = new StructPollfd();
				pollFDs[pollIndex].fd = socketFD;
				pollFDs[pollIndex].events = (short)POLLIN;
				++pollIndex;
			}

			final int usapPoolEventFDIndex = pollIndex;

			if (mUsapPoolEnabled) {
				pollFDs[pollIndex] = new StructPollfd();
				pollFDs[pollIndex].fd = mUsapPoolEventFD;
				pollFDs[pollIndex].events = (short)POLLIN;
				++pollIndex;

				// The usapPipeFDs array will always be filled in if the USAP Pool is enabled.
				assert usapPipeFDs != null;
				for (int usapPipeFD : usapPipeFDs) {
					FileDescriptor managedFd = new FileDescriptor();
					managedFd.setInt$(usapPipeFD);

					pollFDs[pollIndex] = new StructPollfd();
					pollFDs[pollIndex].fd = managedFd;
					pollFDs[pollIndex].events = (short)POLLIN;
					++pollIndex;
				}
			}

			int pollTimeoutMs;

			if (mUsapPoolRefillTriggerTimestamp == INVALID_TIMESTAMP) {
				pollTimeoutMs = -1;
			} else {
				long elapsedTimeMs = System.currentTimeMillis() - mUsapPoolRefillTriggerTimestamp;

				if (elapsedTimeMs >= mUsapPoolRefillDelayMs) {
					// The refill delay has elapsed during the period between poll invocations.
					// We will now check for any currently ready file descriptors before refilling
					// the USAP pool.
					pollTimeoutMs = 0;
					mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
					mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

				} else if (elapsedTimeMs <= 0) {
					// This can occur if the clock used by currentTimeMillis is reset, which is
					// possible because it is not guaranteed to be monotonic.  Because we can't tell
					// how far back the clock was set the best way to recover is to simply re-start
					// the respawn delay countdown.
					pollTimeoutMs = mUsapPoolRefillDelayMs;

				} else {
					pollTimeoutMs = (int)(mUsapPoolRefillDelayMs - elapsedTimeMs);
				}
			}

			int pollReturnValue;
			try {
				pollReturnValue = Os.poll(pollFDs, pollTimeoutMs);
			} catch (ErrnoException ex) {
				throw new RuntimeException("poll failed", ex);
			}

			if (pollReturnValue == 0) {
				// The poll returned zero results either when the timeout value has been exceeded
				// or when a non-blocking poll is issued and no FDs are ready.  In either case it
				// is time to refill the pool.  This will result in a duplicate assignment when
				// the non-blocking poll returns zero results, but it avoids an additional
				// conditional in the else branch.
				mUsapPoolRefillTriggerTimestamp = INVALID_TIMESTAMP;
				mUsapPoolRefillAction = UsapPoolRefillAction.DELAYED;

			} else {
				boolean usapPoolFDRead = false;

				while (--pollIndex >= 0) {
					if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
						continue;
					}

					if (pollIndex == 0) {
						// Zygote server socket
						ZygoteConnection newPeer = acceptCommandPeer(abiList);
						peers.add(newPeer);
						socketFDs.add(newPeer.getFileDescriptor());
					} else if (pollIndex < usapPoolEventFDIndex) {
						// Session socket accepted from the Zygote server socket
						try {
							ZygoteConnection connection = peers.get(pollIndex);
							boolean multipleForksOK = !isUsapPoolEnabled() && ZygoteHooks.isIndefiniteThreadSuspensionSafe();
							final Runnable command = connection.processCommand(this, multipleForksOK);

							// TODO (chriswailes): Is this extra check necessary?
							if (mIsForkChild) {
								// We're in the child. We should always have a command to run at
								// this stage if processCommand hasn't called "exec".
								if (command == null) {
									throw new IllegalStateException("command == null");
								}
								return command;
							} else {
								// We're in the server - we should never have any commands to run.
								if (command != null) {
									throw new IllegalStateException("command != null");
								}

								// We don't know whether the remote side of the socket was closed or
								// not until we attempt to read from it from processCommand. This
								// shows up as a regular POLLIN event in our regular processing
								// loop.
								if (connection.isClosedByPeer()) {
									connection.closeSocket();
									peers.remove(pollIndex);
									socketFDs.remove(pollIndex);
								}
							}
						} catch (Exception e) {
							if (!mIsForkChild) {
								// We're in the server so any exception here is one that has taken
								// place pre-fork while processing commands or reading / writing
								// from the control socket. Make a loud noise about any such
								// exceptions so that we know exactly what failed and why.

								Slog.e(TAG, "Exception executing zygote command: ", e);

								// Make sure the socket is closed so that the other end knows
								// immediately that something has gone wrong and doesn't time out
								// waiting for a response.
								ZygoteConnection conn = peers.remove(pollIndex);
								conn.closeSocket();

								socketFDs.remove(pollIndex);
							} else {
								// We're in the child so any exception caught here has happened post
								// fork and before we execute ActivityThread.main (or any other
								// main() method). Log the details of the exception and bring down
								// the process.
								Log.e(TAG, "Caught post-fork exception in child process.", e);
								throw e;
							}
						} finally {
							// Reset the child flag, in the event that the child process is a child-
							// zygote. The flag will not be consulted this loop pass after the
							// Runnable is returned.
							mIsForkChild = false;
						}

					} else {
						// Either the USAP pool event FD or a USAP reporting pipe.

						// If this is the event FD the payload will be the number of USAPs removed.
						// If this is a reporting pipe FD the payload will be the PID of the USAP
						// that was just specialized.  The `continue` statements below ensure that
						// the messagePayload will always be valid if we complete the try block
						// without an exception.
						long messagePayload;

						try {
							byte[] buffer = new byte[Zygote.USAP_MANAGEMENT_MESSAGE_BYTES];
							int readBytes =
								Os.read(pollFDs[pollIndex].fd, buffer, 0, buffer.length);

							if (readBytes == Zygote.USAP_MANAGEMENT_MESSAGE_BYTES) {
								DataInputStream inputStream =
									new DataInputStream(new ByteArrayInputStream(buffer));

								messagePayload = inputStream.readLong();
							} else {
								Log.e(TAG, "Incomplete read from USAP management FD of size " + readBytes);
								continue;
							}
						} catch (Exception ex) {
							if (pollIndex == usapPoolEventFDIndex) {
								Log.e(TAG, "Failed to read from USAP pool event FD: " + ex.getMessage());
							} else {
								Log.e(TAG, "Failed to read from USAP reporting pipe: " + ex.getMessage());
							}

							continue;
						}

						if (pollIndex > usapPoolEventFDIndex) {
							Zygote.removeUsapTableEntry((int)messagePayload);
						}

						usapPoolFDRead = true;
					}
				}

				if (usapPoolFDRead) {
					int usapPoolCount = Zygote.getUsapPoolCount();

					if (usapPoolCount < mUsapPoolSizeMin) {
						// Immediate refill
						mUsapPoolRefillAction = UsapPoolRefillAction.IMMEDIATE;
					} else if (mUsapPoolSizeMax - usapPoolCount >= mUsapPoolRefillThreshold) {
						// Delayed refill
						mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
					}
				}
			}

			if (mUsapPoolRefillAction != UsapPoolRefillAction.NONE) {
				int[] sessionSocketRawFDs =
					socketFDs.subList(1, socketFDs.size())
						.stream()
						.mapToInt(FileDescriptor::getInt$)
						.toArray();

				final boolean isPriorityRefill =
					mUsapPoolRefillAction == UsapPoolRefillAction.IMMEDIATE;

				final Runnable command =
					fillUsapPool(sessionSocketRawFDs, isPriorityRefill);

				if (command != null) {
					return command;
				} else if (isPriorityRefill) {
					// Schedule a delayed refill to finish refilling the pool.
					mUsapPoolRefillTriggerTimestamp = System.currentTimeMillis();
				}
			}
		}
	}
```

这个里面是看`mIsForkChild`这个属性来进行判断是不是子进程，如果是就返回出去，父进程继续监听，这个属性是在
`ZygoteConnection.java`文件里面被调用的代码过多了，我们就拿具体的一部分来

``` java
pid = Zygote.forkAndSpecialize(parsedArgs.mUid, parsedArgs.mGid, parsedArgs.mGids, parsedArgs.mRuntimeFlags, rlimits, parsedArgs.mMountExternal, parsedArgs.mSeInfo, parsedArgs.mNiceName, fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote, parsedArgs.mInstructionSet, parsedArgs.mAppDataDir, parsedArgs.mIsTopApp, parsedArgs.mPkgDataInfoList, parsedArgs.mAllowlistedDataInfoList, parsedArgs.mBindMountAppDataDirs, parsedArgs.mBindMountAppStorageDirs);

if (pid == 0) {
	// in child
	zygoteServer.setForkChild();

	zygoteServer.closeServerSocket();
	IoUtils.closeQuietly(serverPipeFd);
	serverPipeFd = null;

	return handleChildProc(parsedArgs, childPipeFd, parsedArgs.mStartChildZygote);
} else {
	// In the parent. A pid < 0 indicates a failure and will be handled in
	// handleParentProc.
	IoUtils.closeQuietly(childPipeFd);
	childPipeFd = null;
	handleParentProc(pid, serverPipeFd);
	return null;
}
```

为什么说确定是这里呢？因为我们根据fork的系统机制可以知道，我们得到子进程的判断方法就是`pid == 0`但是整个文件就只有这里符合，所以就确定是这里，然后这个然后我们来看这个内部的子进程控制器

``` Java
private Runnable handleChildProc(ZygoteArguments parsedArgs, FileDescriptor pipeFd, boolean isZygote)
{
	/*
	 * By the time we get here, the native code has closed the two actual Zygote
	 * socket connections, and substituted /dev/null in their place.  The LocalSocket
	 * objects still need to be closed properly.
	 */

	closeSocket();

	Zygote.setAppProcessName(parsedArgs, TAG);

	// End of the postFork event.
	Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
	if (parsedArgs.mInvokeWith != null) {
		WrapperInit.execApplication(parsedArgs.mInvokeWith,
			parsedArgs.mNiceName,
			parsedArgs.mTargetSdkVersion,
			VMRuntime.getCurrentInstructionSet(),
			pipeFd,
			parsedArgs.mRemainingArgs);

		// Should not get here.
		throw new IllegalStateException("WrapperInit.execApplication unexpectedly returned");
	} else {
		if (!isZygote) {
			return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
				parsedArgs.mDisabledCompatChanges,
				parsedArgs.mRemainingArgs,
				null /* classLoader */);
		} else {
			return ZygoteInit.childZygoteInit( parsedArgs.mRemainingArgs /* classLoader */);
		}
	}
}
```

这个会走最下面的`childZygoteInit`方法然后这个方法会去查找它的静态main方法调用`findStaticMain`方法

``` Java
protected static Runnable findStaticMain(String className, String[] argv, ClassLoader classLoader)
{
	Class<?> cl;

	try {
		cl = Class.forName(className, true, classLoader);
	} catch (ClassNotFoundException ex) {
		throw new RuntimeException(
			"Missing class when invoking static main " + className,
			ex);
	}

	Method m;
	try {
		m = cl.getMethod("main", new Class[] { String[].class });
	} catch (NoSuchMethodException ex) {
		throw new RuntimeException(
			"Missing static main on " + className,
			ex);
	} catch (SecurityException ex) {
		throw new RuntimeException(
			"Problem getting static main on " + className,
			ex);
	}

	int modifiers = m.getModifiers();
	if (!(Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
		throw new RuntimeException(
			"Main method is not public and static on " + className);
	}

	/*
	 * This throw gets caught in ZygoteInit.main(), which responds
	 * by invoking the exception's run() method. This arrangement
	 * clears up all the stack frames that were required in setting
	 * up the process.
	 */
	return new MethodAndArgsCaller(m, argv);
}
```

然后返回的对象是一个自定义的对象

``` Java
static class MethodAndArgsCaller implements Runnable
{
	/** method to call */
	private final Method mMethod;

	/** argument array */
	private final String[] mArgs;

	public MethodAndArgsCaller(Method method, String[] args)
	{
		mMethod = method;
		mArgs = args;
	}

	public void run()
	{
		try {
			mMethod.invoke(null, new Object[] { mArgs });
		} catch (IllegalAccessException ex) {
			throw new RuntimeException(ex);
		} catch (InvocationTargetException ex) {
			Throwable cause = ex.getCause();
			if (cause instanceof RuntimeException) {
				throw (RuntimeException)cause;
			} else if (cause instanceof Error) {
				throw (Error)cause;
			}
			throw new RuntimeException(ex);
		}
	}
}
```

这时候，参数和启动main方法都集齐了，就可以返回到最顶层进行调用了，所以我们会看见方法结束时候会调用`caller.run()`

``` java
if (caller != null) {
	caller.run(); 
}
```

然后就走到头了，这个就是具体的内部方法，好像有点跑偏了，想说fork的结果成怎么启动了



## UID的分配
这个其实没啥说的，`UID`叫作`User ID`就是他所归用户的一个索引，用来鉴权和隔离资源等等操作

``` text
父进程 UID=system (1000)
     |
  fork()
     |
子进程 UID=system (1000)
```

内核不会自动给子进程分配新的 UID，是通过`forkAndSpecialize`函数进行fork和分配的

``` java 
static int forkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags, int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose, int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir, boolean isTopApp, String[] pkgDataInfoList, String[] allowlistedDataInfoList, boolean bindMountAppDataDirs, boolean bindMountAppStorageDirs)
{
	ZygoteHooks.preFork();

	int pid = nativeForkAndSpecialize(
		uid,
		gid,
		gids,
		runtimeFlags,
		rlimits,
		mountExternal,
		seInfo,
		niceName,
		fdsToClose,
		fdsToIgnore,
		startChildZygote,
		instructionSet,
		appDataDir,
		isTopApp,
		pkgDataInfoList,
		allowlistedDataInfoList,
		bindMountAppDataDirs,
		bindMountAppStorageDirs);
	if (pid == 0) {
		// Note that this event ends at the end of handleChildProc,
		Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "PostFork");

		// If no GIDs were specified, don't make any permissions changes based on groups.
		if (gids != null && gids.length > 0) {
			NetworkUtilsInternal.setAllowNetworkingForProcess(containsInetGid(gids));
		}
	}

	// Set the Java Language thread priority to the default value for new apps.
	Thread.currentThread().setPriority(Thread.NORM_PRIORITY);

	ZygoteHooks.postForkCommon();
	return pid;
}
private static native int nativeForkAndSpecialize(int uid, int gid, int[] gids, int runtimeFlags, int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose, int[] fdsToIgnore, boolean startChildZygote, String instructionSet, String appDataDir, boolean isTopApp, String[] pkgDataInfoList, String[] allowlistedDataInfoList, boolean bindMountAppDataDirs, boolean bindMountAppStorageDirs);
```

这个函数的参数很多，不过也不用全看，只看我们关心的就可以，但是我们看见这里面并没有我们想要的，那我们只能去追下面的函数了，直接说结论吧，如果想找以后在模块原理会看
- nativeForkAndSpecialize：调用Linux的fork()子进程，设置新进程的主线程id，重置gc性能数据，设置信号处理函数等功能
## 总结
差不多两万字，我感觉代码有一万五，我自己写的很少，我也推荐各位自己追一手这个启动流程，抽丝剥茧的感觉还是挺好的
---
title: 理解Android进程的创建流程
tags:
---


# ActivityManagerService#startProcess

```
private ProcessStartResult startProcess(String hostingType, String entryPoint,
        ProcessRecord app, int uid, int[] gids, int runtimeFlags, int mountExternal,
        String seInfo, String requiredAbi, String instructionSet, String invokeWith,
        long startTime) {
    
        startResult = Process.start(entryPoint,
                app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                app.info.dataDir, invokeWith,
                new String[] {PROC_START_SEQ_IDENT + app.startSeq});
        }
}
```

# Process

## start
```
public static final Process.ProcessStartResult start(String processClass, String niceName, int uid, int gid, int[] gids, int debugFlags, int targetSdkVersion, String[] zygoteArgs) {
    try {
        return startViaZygote(processClass, niceName, uid, gid, gids, debugFlags, targetSdkVersion, zygoteArgs);
    } catch (ZygoteStartFailedEx var9) {
        Log.e("Process", "Starting VM process through Zygote failed");
        throw new RuntimeException("Starting VM process through Zygote failed", var9);
    }
}
```

## startViaZygote

```
    private static Process.ProcessStartResult startViaZygote(String processClass, String niceName, int uid, int gid, int[] gids, int debugFlags, int targetSdkVersion, String[] extraArgs) throws ZygoteStartFailedEx {
        Class var8 = Process.class;
        synchronized(Process.class) {
            ArrayList<String> argsForZygote = new ArrayList();
            argsForZygote.add("--runtime-init");
            argsForZygote.add("--setuid=" + uid);
            argsForZygote.add("--setgid=" + gid);
            ....
            argsForZygote.add("--target-sdk-version=" + targetSdkVersion);
           ...
            ...
            return zygoteSendArgsAndGetResult(argsForZygote);
        }
    }

```

该过程主要工作是生成argsForZygote数组，该数组保存了进程的uid、gid、groups、target-sdk、nice-name等一系列的参数。

## zygoteSendArgsAndGetResult

```
    private static Process.ProcessStartResult zygoteSendArgsAndGetResult(ArrayList<String> args) throws ZygoteStartFailedEx {
        openZygoteSocketIfNeeded();

        try {
            sZygoteWriter.write(Integer.toString(args.size()));
            sZygoteWriter.newLine();
            int sz = args.size();

            for(int i = 0; i < sz; ++i) {
                String arg = (String)args.get(i);
                if (arg.indexOf(10) >= 0) {
                    throw new ZygoteStartFailedEx("embedded newlines not allowed");
                }

                sZygoteWriter.write(arg);
                sZygoteWriter.newLine();
            }

            sZygoteWriter.flush();
            Process.ProcessStartResult result = new Process.ProcessStartResult();
            //等待socket服务端（即zygote）返回新创建的进程pid;
            //对于等待时长问题，Google正在考虑此处是否应该有一个timeout，但目前是没有的。
            result.pid = sZygoteInputStream.readInt();
            if (result.pid < 0) {
                throw new ZygoteStartFailedEx("fork() failed");
            } else {
                result.usingWrapper = sZygoteInputStream.readBoolean();
                return result;
            }
        } catch (IOException var5) {
            try {
                if (sZygoteSocket != null) {
                    sZygoteSocket.close();
                }
            } catch (IOException var4) {
                Log.e("Process", "I/O exception on routine close", var4);
            }

            sZygoteSocket = null;
            throw new ZygoteStartFailedEx(var5);
        }
    }
```
system_server 进程和 Zygote 进程之间是通过 Socket 通信的，这个方法的主要功能是通过socket通道向Zygote进程发送一个参数列表，然后进入阻塞等待状态，直到远端的socket服务端发送回来新创建的进程pid才返回。

既然system_server进程的zygoteSendArgsAndGetResult()方法通过socket向Zygote进程发送消息，这是便会唤醒Zygote进程，来响应socket客户端的请求（即system_server端），接下来的操作便是在Zygote来创建进程


# Zygote创建进程


# Zygote.main

```
    public static void main(String argv[]) {
        ZygoteServer zygoteServer = new ZygoteServer();

        
        // The select loop returns early in the child process after a fork and
        // loops forever in the zygote.
        caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with exception", ex);
            throw ex;
        } finally {
            zygoteServer.closeServerSocket();
        }

        // We're in the child process and have exited the select loop. Proceed to execute the
        // command.
        if (caller != null) {
            caller.run();
        }
    }

```

## 

```
    Runnable runSelectLoop(String abiList) {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
    //sServerSocket是socket通信中的服务端，即zygote进程。保存到fds[0]
        fds.add(mServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
            try {
                //处理轮询状态，当pollFds有事件到来则往下执行，否则阻塞在这里
                Os.poll(pollFds, -1);
            } catch (ErrnoException ex) {
                throw new RuntimeException("poll failed", ex);
            }
            //采用I/O多路复用机制，当接收到客户端发出连接请求 或者数据处理请求到来，则往下执行；
            // 否则进入continue，跳出本次循环。
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }

                if (i == 0) {
                     //即fds[0]，代表的是sServerSocket，则意味着有客户端连接请求；
                // 则创建ZygoteConnection对象,并添加到fds。//【见小节5.1】
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    try {
                        ZygoteConnection connection = peers.get(i);
                        final Runnable command = connection.processOneCommand(this);

                        if (mIsForkChild) {
                            // We're in the child. We should always have a command to run at this
                            // stage if processOneCommand hasn't called "exec".
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
                            // not until we attempt to read from it from processOneCommand. This shows up as
                            // a regular POLLIN event in our regular processing loop.
                            if (connection.isClosedByPeer()) {
                                connection.closeSocket();
                                peers.remove(i);
                                fds.remove(i);
                            }
                        }
                    } catch (Exception e) {
                        if (!mIsForkChild) {
                            // We're in the server so any exception here is one that has taken place
                            // pre-fork while processing commands or reading / writing from the
                            // control socket. Make a loud noise about any such exceptions so that
                            // we know exactly what failed and why.

                            Slog.e(TAG, "Exception executing zygote command: ", e);

                            // Make sure the socket is closed so that the other end knows immediately
                            // that something has gone wrong and doesn't time out waiting for a
                            // response.
                            ZygoteConnection conn = peers.remove(i);
                            conn.closeSocket();

                            fds.remove(i);
                        } else {
                            // We're in the child so any exception caught here has happened post
                            // fork and before we execute ActivityThread.main (or any other main()
                            // method). Log the details of the exception and bring down the process.
                            Log.e(TAG, "Caught post-fork exception in child process.", e);
                            throw e;
                        }
                    } finally {
                        // Reset the child flag, in the event that the child process is a child-
                        // zygote. The flag will not be consulted this loop pass after the Runnable
                        // is returned.
                        mIsForkChild = false;
                    }
                }
            }
        }
    }

```


# 总结
system_server进程等待zygote返回进程创建完成(ZygoteConnection.handleParentProc), 一旦Zygote.forkAndSpecialize()方法执行完成, 那么分道扬镳, zygote告知system_server进程进程已创建, 而子进程继续执行后续的handleChildProc操作.




---

logcat小技巧
通过adb bugreport抓取log信息.先看zygote是否起来, 再看system_server主线程的运行情况,再看ActivityManager情况

```
adb logcat -s Zygote
adb logcat -s SystemServer
adb logcat -s SystemServiceManager
adb logcat | grep "1359 1359" //system_server情况
adb logcat -s ActivityManager
```
现场调试命令
```
cat proc/[pid]/stack ==> 查看kernel调用栈
debuggerd -b [pid] ==> 也不可以不带参数-b, 则直接输出到/data/tombstones/目录
kill -3 [pid] ==> 生成/data/anr/traces.txt文件
lsof [pid] ==> 查看进程所打开的文件
```

---

进程|	主方法
|--|--
init进程	|Init.main()
zygote进程|	ZygoteInit.main()
app_process进程|	RuntimeInit.main()
system_server进程|	SystemServer.main()
app进程|	ActivityThread.main()


注意app_process进程是指通过/system/bin/app_process启动的进程，且后面跟的参数不带–zygote，即并非启动zygote进程。 比如常见的有通过adb shell方式来执行am,pm等命令，便是这种方式。
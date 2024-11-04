# Windows Driver -- Windbg联调环境配置

参考：https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-a-network-debugging-connection-automatically

## Target PC配置

注意：

Target PC要先开测试模式，且关闭系统所有网络防火墙。

Target PC要能Ping通Host PC，即Host PC和Target PC可以用同一局域网的路由器(Router)或交换机(Hub)的网口相连。

Target PC的kdnet创建一次以后再创建也不会变。

Kdnet配置完成后，设备管理器的网络设备会有Kernel Debug Network设备（KDNET），且网口设备（例如下面的Intel Ethernet）会有感叹号Code53是正常的，表示此网口正作为KDNET debug端口。

```
C:\Windows\System32>
C:\Windows\System32>cd c:\kdnet

c:\kdnet>kdnet.exe

Network debugging is supported on the following NICs:
busparams=0.31.6, Intel(R) Ethernet Connection (7) I219-V, KDNET is running on this NIC.

Network debugging is supported on the following USB controllers:
busparams=0.20.0, Intel(R) USB 3.1 eXtensible Host Controller - 1.10 (Microsoft)

This Microsoft hypervisor supports using KDNET in guest VMs.

c:\kdnet>
c:\kdnet>kdnet.exe 10.52.4.41 50000

Enabling network debugging on Intel(R) Ethernet Connection (7) I219-V.

To debug this machine, run the following command on your debugger host machine.
windbg -k net:port=50000,key=3s1m4bjm7ihi7.3b8dig9hl019g.2fyvva9v1ie3a.38mfaw4eweiuo

Then reboot this machine by running shutdown -r -t 0 from this command prompt.

c:\kdnet>ping 10.52.4.41

Pinging 10.52.4.41 with 32 bytes of data:
Reply from 10.52.4.41: bytes=32 time=4ms TTL=128
Reply from 10.52.4.41: bytes=32 time=1ms TTL=128
Reply from 10.52.4.41: bytes=32 time=1ms TTL=128
Reply from 10.52.4.41: bytes=32 time=1ms TTL=128

Ping statistics for 10.52.4.41:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 1ms, Maximum = 4ms, Average = 1ms
```

## Host PC配置

Host PC也关闭系统网络防火墙。

用Microsoft Store安装的Windbg Preview，Attach to Kernel，填入Target PC生成的Port和Key：

```
port=50000,key=3s1m4bjm7ihi7.3b8dig9hl019g.2fyvva9v1ie3a.38mfaw4eweiuo
```

连接成功后显示Connected to target ....

```
Using NET for debugging
Opened WinSock 2.0
Waiting to reconnect...

Connected to target 10.52.5.0 on port 50000 on local IP 10.52.4.41.
```

## Windbg联机观测DbgPrint

### Windbg显示DbgPrint

需要打开Target PC的“Enabling verbose kernel output”

可用方法：

（1）在Target PC安装DebugView( [Sysinternals DebugView](https://docs.microsoft.com/en-us/sysinternals/downloads/debugview))，需要运行DebugView，选项勾选“Enable Kernel Debug”和“Enabling verbose kernel output”，才可以在DebugView和Host PC的Windbg同时观测到DbgPrint的打印。

（2）或者在Target PC安装DebugLogger([DebugLogger](https://github.com/tandasat/DebugLogger), open source implementation of [Sysinternals DebugView](https://docs.microsoft.com/en-us/sysinternals/downloads/debugview))，安装后默认打开了“Enabling verbose kernel output”，无需运行DebugLogger就可以在Host PC的Windbg观测到DebugPrint打印。这种方式可以记录系统重启中的Kernel过程。

注意：以下方法只是在Target PC手动设置打开所有过滤级别，并不能在Host PC的Windbg观测到Target PC的DebugPrint打印：

[Reading and Filtering Debugging Messages](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/reading-and-filtering-debugging-messages)

### Windbg记录DbgPrint到log文件：

Command -> Save Window to File。使用Log中的特定关键字过滤，另存为Filtered log再查看

## Windbg的联机调试方法

[Debug Windows drivers step-by-step lab (echo kernel mode)](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/debug-universal-drivers---step-by-step-lab--echo-kernel-mode-)
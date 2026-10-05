<a name="chinese"></a>

**简体中文** | [English](#english)

# UnrealVTDbg

**基于 Intel VT-x / EPT 的 Windows 内核调试支撑体系（原始版本代码）**

状态：原始版本代码（本仓库未做完整构建与真机验证） · 平台：Windows 10 / Windows 11 x64 · 依赖：Intel VT-x + EPT

> 已知问题：Intel Core Ultra 等 P 核 + E 核混合架构需要在 EPT 初始化时按逻辑处理器适配能力，
> 详见 [Intel Core Ultra 大小核（P/E）的 EPT 适配](#hybrid-ept)。

UnrealVTDbg（代码内的项目名为 UnrealDbg）是一套自建调试体系：以内核 VT-x Hypervisor 为核心，
通过 EPT 钩子、无痕断点和对 Dbgk（Windows 内核调试子系统）的接管，把常规用户态调试器接到受保护进程上，
用于对抗驱动侧的反调试检测。项目最初面向 Windows 10 / Windows 11 x64，与具体系统版本的内核符号、
结构偏移和特征码强相关。

> 范围说明：本文只描述 C++ / 驱动部分，即 `VT_Driver`、`DbgkSysWin10`、`DbgkSysWin11`、
> `UnrealDbgDll`、`Hook`、`Loader` 和 `Common`。仓库中的 Delphi 前端与配套工具
> （`UnrealDbg/`、`CardRegistration/`、`D-encryption/`、`SymbolTool/`）不在本文范围内。
>
> 本仓库是原始代码整理版，不保证能直接编译或运行。构建与运行前请先阅读
> [已知限制](#已知限制) 和 [免责声明](#免责声明)。

## 核心能力

- **VT-x Hypervisor 宿主**：VMXON / VMLAUNCH、VMCS 管理、VM-exit 分发、VMCALL 网关，
  按逻辑处理器维护 vCPU 上下文。
- **EPT 内存虚拟化**：EPT 页表与影子页管理，支持 EPT 执行钩子、内存读写执行监视、
  假页（fake page）内存读取以及 EPT 函数钩子。
- **无痕断点**：隐藏软件断点（0xCC）的实际字节、读取被隐藏的断点内容、保护调试寄存器（DRx），
  让目标进程和反调试驱动看不到断点痕迹。
- **VMCALL 控制接口**：内置 21 个功能号，覆盖 VMXOFF、INVEPT、EPT 钩子/摘钩、
  隐藏 Hypervisor 存在、软件断点隐藏与读取、EPT 读/写/执行监视、VMCS 状态导出等。
- **Dbgk 接管**：解析 `ntoskrnl`、`win32kbase`、`win32kfull` 的符号，
  以 EPT 函数钩子接管 Dbgk / Psp 内部例程，重定向调试对象与调试事件流。
- **调试器侧 Hook DLL**：注入第三方调试器进程，Hook `NtDebugActiveProcess`、
  `WaitForDebugEvent`、`ContinueDebugEvent`、`ReadProcessMemory`、`WriteProcessMemory` 等调试 API。
- **内核级断点管理**：通过 IOCTL 设置与删除硬件断点（DRx）和软件断点，断点状态可由 VT 层隐藏。
- **符号加载器**：自动下载 `ntoskrnl.exe`、`win32kbase.sys`、`win32kfull.sys` 的 PDB 符号
  到 `C:\Symbols\`，供驱动解析内核内部符号使用。
- **驱动加载封装**：`AIHelper.dll` 统一提供驱动安装/卸载与 DLL 注入能力
  （`LoadNT` / `UnloadNT` / `InjectDll` / `outDebug`）。

## 系统架构

```text
+---------------------------------------------------------------+
|              Delphi 前端 / 外部宿主（原始配套，略）             |
+-------------------------------+-------------------------------+
                                | 调用导出接口
+-------------------------------v-------------------------------+
| UnrealDbgDll.dll（Ring3 调度层）                               |
|   Initialize / StartProcess / GetFileVersion                  |
|   TL_BlockGameResumeThread                                    |
+---------------+----------------------------+------------------+
                |                            |
                | AIHelper!LoadNT            | CreateProcess + InjectDll
                v                            v
+-----------------------------+   +------------------------------+
| VT_Driver.sys（Hypervisor） |   | Hook.dll / Hook64.dll         |
|   VMXON / VMCS / EPT        |<--|  调试 API Hook + VMCALL 包装   |
|   VM-exit / VMCALL 分发     |   |  Detours + EPT 钩子           |
+--------------+--------------+   +---------------+--------------+
               ^                                  |
               | VMCALL（DPC 广播到各逻辑处理器）    | IOCTL \\.\UnrealDbg
+--------------+----------------------------------v--------------+
| DbgkSysWin10.sys / DbgkSysWin11.sys（调试支撑驱动）             |
|   符号解析 / Dbgk 钩子 / 断点管理 / 密钥校验 / 调试数据转发      |
+----------------------------------------------------------------+
```

分层职责：

| 层 | 组件 | 运行位置 | 职责 |
|---|---|---|---|
| Hypervisor | `VT_Driver.sys` | Ring 0 | 在 Intel VT-x 下运行原系统，提供 EPT 钩子与 VMCALL 服务 |
| 调试支撑驱动 | `DbgkSysWin10/11.sys` | Ring 0 | 解析符号、接管 Dbgk、管理断点、加解密 IOCTL 数据 |
| 调度层 | `UnrealDbgDll.dll` | Ring 3 | 装载驱动、下发密钥与调试数据、拉起并注入调试器 |
| 调试器侧 | `Hook.dll` / `Hook64.dll` | 目标调试器进程 | Hook 调试 API，通过 IOCTL / VMCALL 协调驱动 |
| 辅助层 | `Loader`（`VTDebugger.exe`） | Ring 3 | 符号下载与管理 |
| 公共库 | `Common/` | 各层共用 | Detours、加密、日志、PDB 解析、共享结构 |

## 仓库结构

```text
.
├─ UnrealDbg.sln              # VS 解决方案（6 个 C++ 工程）
├─ VT_Driver/                 # VT-x Hypervisor 宿主驱动（C++ / ASM，WDM）
├─ DbgkSysWin10/              # Windows 10 调试支撑驱动（WDM）
├─ DbgkSysWin11/              # Windows 11 调试支撑驱动（WDM）
├─ UnrealDbgDll/              # Ring3 调度 DLL（导出接口）
├─ Hook/                      # 注入调试器的 Hook DLL（x86 / x64）
├─ Loader/                    # 符号加载器，产物 VTDebugger.exe
├─ Common/                    # 公共库（Detours、加密、日志、PDB 解析等）
├─ _deps/                     # 预编译辅助组件（AIHelper、VMProtect 运行库等）
├─ Common/VT_Driver/          # 驱动侧共用的 VT/EPT 支撑代码副本
├─ UnrealDbg/                 # 原始 Delphi 主界面（不在本文范围）
├─ CardRegistration/          # 卡密注册工具（Delphi，不在本文范围）
├─ D-encryption/              # 加密工具（Delphi，不在本文范围）
├─ D-encryptiondll/           # 加密 DLL 源码（配套工具）
└─ SymbolTool/                # 符号工具（Delphi，不在本文范围）
```

## 模块详解

### 1. VT_Driver（Hypervisor 宿主）

`DriverEntry`（`VT_Driver/Driver.cpp`）的执行顺序：

1. 调用 `InitNtoskrnlSymbolsTable()` 解析 `ntoskrnl` 符号。
2. 调用 `hv::virtualization_support()` 检查 CPU 是否支持 VMX 操作。
3. 调用 `hv::InitGlobalVariables()` 初始化全局状态，再由 `vmm_init()` 逐处理器启动 VT。
4. 失败时执行 `hv::disable_vmx_operation()` 并释放 vCPU 上下文。

`Unload` 会先摘除全部 EPT 钩子（`hvgt::ept_unhook()`），再对每个逻辑处理器执行
`hvgt::vmoff()`，最后关闭 VMX 并释放资源。

关键文件：

| 文件 | 说明 |
|---|---|
| `Driver.cpp` | 驱动入口/出口，初始化 VT 与符号表 |
| `vmm.cpp` / `vmcs.cpp` | vCPU 管理、VMCS 字段读写与初始化 |
| `vmexit_handler.cpp` | VM-exit 分发：指令调整、异常注入、EPT 处理等 |
| `vmcall_handler.cpp` | VMCALL 功能号分发与参数解析 |
| `EPT.cpp` | EPT 页表、钩子页、内存监视与假页实现 |
| `hypervisor_gateway.cpp` | 宿主侧封装：EPT 钩子、INVEPT、VMXOFF、隐藏 Hypervisor 等 |
| `ASM/*.asm` | VMX 指令封装、VM-exit 入口、中断处理与 LDE64 长度反汇编 |
| `crx.h` / `drx.h` / `msr.h` / `mtrr.h` | 控制寄存器、调试寄存器、MSR 与 MTRR 支持 |

说明：当前实现只针对 Intel VT-x（`vmx.h` / `VMX_*`），不包含 AMD-V 路径。

### 2. DbgkSysWin10 / DbgkSysWin11（调试支撑驱动）

两个目录源码结构基本一致（各 62 个源文件），差异集中在系统版本相关的符号、结构体偏移和特征码。
`UnrealDbgDll` 根据系统 Build Number 选择装载：`>= 22000` 使用 Win11 版本，否则使用 Win10 版本。

设备接口（两个版本相同）：

- 设备名：`\Device\DbgkSysDevice`
- 符号链接：`\??\UnrealDbg`，用户态路径为 `\\.\UnrealDbg`
- 缓冲方式：`DO_BUFFERED_IO`，`IRP_MJ_DEVICE_CONTROL` 由 `HandlerDispatchRoutin` 处理

主要机制：

1. **加密的 IOCTL 数据**：用户态把明文（`USER_DATA` 结构）用 Blowfish 加密后下发，
   驱动解密并校验长度/内容；本仓库中的固定密钥校验成功后会回写魔数 `1998`。
2. **符号解析**：密钥校验通过后依次执行 `InitNtoskrnlSymbolsTable()`、
   `InitWin32kbaseSymbolsTable()`、`InitWin32kfullSymbolsTable()`，解析内核内部符号，
   例如 `Sys_DbgkCreateThread`、`Sys_PspExitThread`、`Sys_DbgkExitThread`、
   `DbgkDebugObjectType`、`Sys_MmProtectVirtualMemory` 等。
3. **偏移下发**：`DispatchOffsetToHost()` 通过 `VMCALL_INIT_OFFSET` 把驱动得到的内核结构偏移
   （如 `ETHREAD.Cid`）下发给 VT 宿主。
4. **Dbgk 初始化**：调用 `DbgkInitialize()` 与 `SetupEptHook()` 安装接管逻辑。
5. **EPT 函数钩子**：通过 `hvgt::hook_function()`（内部广播 `VMCALL_EPT_CC_HOOK`）挂钩
   `NtTerminateProcess`、`PspExitThread`、`PspCreateThread` 等内核例程。
   原始代码中 `SetupHook_DbgkCreateThread_CMP_Debugport()` 处于注释状态，
   `SetupHook_PspExitThread_CMP_Debugport()` 已启用。
6. **保护与对抗模块**：`Protect/` 下包含进程保护、线程 DRx 保护、
   `BypassFindWnd`（绕过窗口查找）等逻辑；`DbgkApi/` 提供调试对象与 Dbgk 消息处理。

`ntos/inc/10.0.19041.4291/` 说明 Win10 版本以 19041.4291 的内核头文件为基线，
其余内部结构通过符号加特征码定位。系统更新后符号与代码特征变化会导致初始化失败。

### 3. UnrealDbgDll（Ring3 调度层）

导出接口（`UnrealDbgDll/dllexport.def`）：

| 导出 | 说明 |
|---|---|
| `Initialize(key)` | 装载驱动、下发密钥、初始化符号表并注册当前调试器进程 |
| `StartProcess(szExe, sPath)` | 创建目标调试器进程，注入 `Hook/Hook64`，上报调试器信息 |
| `GetFileVersion` | 读取文件版本信息 |
| `TL_BlockGameResumeThread(pid)` | 通过 IOCTL 阻止目标进程的恢复线程 |

`Initialize` 的实际流程：

1. `_Initialize(模块目录)` 通过 `AIHelper.dll` 的 `LoadNT` 依次装载
   `VT_Driver.sys`（服务名 `VT_Driver`）和 `DbgkSysWin10.sys` / `DbgkSysWin11.sys`
   （服务名 `UnrealDevice`），随后打开 `\\.\UnrealDbg` 设备。
2. `LoadSymbolsTable(key)` 通过 `IOCTL_LOAD_SYMBOLS_TABLE` 下发加密的 `RING3_VERIFY` 结构，
   等待驱动返回成功标志 `1998`。
3. 成功后 `SendDebuggerDataToDriver(GetCurrentProcessId())` 通过
   `IOCTL_LOAD_DEBUGGER_DATA` 注册当前调试器进程。
4. 关键校验逻辑包裹在 `VMProtectBeginVirtualization` / `VMProtectEnd` 中，
   链接 `Common/VMProtect/VMProtectSDK64.lib`。

`StartProcess` 的实际流程：

1. `CreateProcess` 拉起指定的调试器可执行文件。
2. 用 `IsWow64Process` 判断位数，选择 `Hook64.dll` 或 `Hook.dll`。
3. 通过 `AIHelper.dll` 的 `InjectDll` 注入 Hook DLL。
4. `SendDebuggerDataToDriver(pid)` 把调试器进程登记到驱动。

注意：`DispatchSymbol()` 目前为空函数（旧的上报路径已注释），
当前使用的是 `Initialize` 中的密钥校验加调试器登记路径。

### 4. Hook / Hook64（调试器侧 Hook DLL）

注入调试器进程后，`DllMain` 在 `DLL_PROCESS_ATTACH` 时执行：

```text
InitGlobalVariables() -> InitFunction() -> SetupHook()
```

`SetupHook()` 连接 `\\.\UnrealDbg` 设备后安装以下 Detours 钩子
（Hook 侧与驱动侧协同，部分能力最终由 EPT 实现）：

| 类别 | 被 Hook 的函数 |
|---|---|
| 调试接管 | `NtDebugActiveProcess`、`DbgUiIssueRemoteBreakin` |
| 调试事件 | `WaitForDebugEvent`、`ContinueDebugEvent` |
| 输出通道 | `OutputDebugStringA`、`OutputDebugStringW` |
| 线程上下文 | `GetThreadContext`、`SetThreadContext` |
| 内存操作 | `ReadProcessMemory`、`WriteProcessMemory`、`VirtualProtectEx` |

`UnHook()` 中另有 `DebugActiveProcess`、`DbgUiDebugActiveProcess`、`NtCreateUserProcess`
等解除挂钩逻辑，属于预留或历史路径。

其他文件：

| 路径 | 说明 |
|---|---|
| `vmx/vmx.cpp` | 用户态 VMCALL 包装：`__vm_call`、`vmcall`、`vmcall2`、`current_vmcall` |
| `DebugEvent/` | 调试事件接管与转换 |
| `Inject/ApcInject/` | APC 注入实现 |
| `Inject/ShellCode/` | ShellCode 注入实现 |
| `Channels/` | 与驱动通信的数据封装 |
| `HookCallSet/functionSet.cpp` | 各 Hook 函数的实现集合 |

### 5. Loader（VTDebugger.exe）

`Loader/WinMain.cpp` 是一个带进度显示的符号下载器：

- 启动时询问是否跳过符号下载；
- 依次处理 `ntoskrnl.exe`、`win32kbase.sys`、`win32kfull.sys`；
- 通过 URLMON 的 `IBindStatusCallback` 回调报告进度，把 PDB 下载到 `C:\Symbols\`；
- 下载结果供 `DbgkSysWin10/11` 驱动解析内核内部符号使用。

工程 `TargetName` 为 `VTDebugger`。

### 6. Common（公共库）

| 目录 | 说明 |
|---|---|
| `Shared/` | `SharedStruct.h`、`IOCTLs.h` 等跨层通信契约 |
| `Detours/` | Detours 头文件与 x86/x64 静态库、Hook 封装 |
| `Encrypt/Blowfish/` | Blowfish 加解密实现 |
| `Hash/` | MD5、CRC32 |
| `FileSystem/` | 文件与路径工具 |
| `Logger/` | 日志组件 |
| `IPC/SharedMemory/` | 共享内存通信 |
| `Ring0/SymbolicAccess/` | 驱动侧 PDB 解析、符号提取、Phnt 头与 ia32-doc 定义 |
| `VMProtect/` | VMProtect SDK 头文件与 x86/x64 静态库 |
| `VT_Driver/` | 驱动侧共用的 VT/EPT 代码副本 |

## 通信接口

### VMCALL（Guest 到 Hypervisor）

功能号定义见 `VT_Driver/vmcall_reason.h`，由 VT 宿主在 `vmcall_handler.cpp` 中分发：

| ID | 名称 | 用途 |
|---|---|---|
| 0 | `VMCALL_TEST` | 测试 VMCALL 通道 |
| 1 | `VMCALL_VMXOFF` | 关闭 VMX |
| 2 | `VMCALL_EPT_CC_HOOK` | 基于 EPT 的 0xCC 执行钩子 |
| 3 | `VMCALL_EPT_INT1_HOOK` | 基于 EPT 的 INT1 钩子 |
| 4 | `VMCALL_EPT_RIP_HOOK` | 基于 EPT 的 RIP 重定向钩子 |
| 5 | `VMCALL_EPT_HOOK_FUNCTION` | 函数级 EPT 钩子 |
| 6 | `VMCALL_EPT_UNHOOK_FUNCTION` | 卸载函数级 EPT 钩子 |
| 7 | `VMCALL_INVEPT_CONTEXT` | 失效 EPT TLB 上下文 |
| 8 | `VMCALL_DUMP_POOL_MANAGER` | 导出池管理器信息 |
| 9 | `VMCALL_DUMP_VMCS_STATE` | 导出 VMCS 状态 |
| 10 | `VMCALL_HIDE_HV_PRESENCE` | 隐藏 Hypervisor 存在（CPUID） |
| 11 | `VMCALL_UNHIDE_HV_PRESENCE` | 恢复 Hypervisor 可见 |
| 12 | `VMCALL_HIDE_SOFTWARE_BREAKPOINT` | 隐藏软件断点 |
| 13 | `VMCALL_READ_SOFTWARE_BREAKPOINT` | 读取被隐藏的软件断点内容 |
| 14 | `VMCALL_READ_EPT_FAKE_PAGE_MEMORY` | 读取 EPT 假页原始内存 |
| 15 | `VMCALL_WATCH_WRITES` | EPT 写监视 |
| 16 | `VMCALL_WATCH_READS` | EPT 读监视 |
| 17 | `VMCALL_WATCH_EXECUTES` | EPT 执行监视 |
| 18 | `VMCALL_WATCH_DELETE` | 删除监视项 |
| 19 | `VMCALL_GET_BREAKPOINT` | 获取命中的断点信息 |
| 20 | `VMCALL_INIT_OFFSET` | 下发宿主需要的结构偏移 |

### IOCTL（Ring3 到 DbgkSys）

定义见 `Common/Shared/IOCTLs.h`，`UnrealDbg/Common/Shared/` 下有内容一致的副本，
全部使用 `FILE_DEVICE_UNKNOWN` + `METHOD_BUFFERED` + `FILE_ANY_ACCESS`。
表中的 `0x800` 到 `0x80B` 是 CTL_CODE 的功能号，实际控制码还要叠加设备类型等位
（例如功能号 `0x800` 生成的控制码为 `0x222000`）：

| 功能号 | 名称 | 实现状态 |
|---|---|---|
| `0x800` | `IOCTL_LOAD_SYMBOLS_TABLE` | 已实现：解密、密钥校验、解析符号并初始化 |
| `0x801` | `IOCTL_LOAD_DEBUGGER_STATE` | 预留，当前分支为空 |
| `0x802` | `IOCTL_LOAD_PROTECT_OBJ_DATA` | 预留，当前分支为空 |
| `0x803` | `IOCTL_LOAD_DEBUGGER_DATA` | 已实现：登记调试器进程 |
| `0x804` | `IOCTL_CREATE_REMOTE_THREAD` | 已实现：内核创建远程线程 |
| `0x805` | `IOCTL_SET_HARDWARE_BREAKPOINT` | 已实现：设置硬件断点 |
| `0x806` | `IOCTL_GET_PROCESS_INFO` | 已实现：查询目标进程信息 |
| `0x807` | `IOCTL_TL_BLOCK_RESUME_THREAD` | 已实现：阻止游戏恢复线程 |
| `0x808` | `IOCTL_DEL_HARDWARE_BREAKPOINT` | 已实现：删除硬件断点 |
| `0x809` | `IOCTL_SET_SOFTWARE_BREAKPOINT` | 已实现：设置软件断点 |
| `0x80A` | `IOCTL_DEL_SOFTWARE_BREAKPOINT` | 已实现：删除软件断点 |
| `0x80B` | `IOCTL_READ_SOFTWARE_BREAKPOINT` | 已实现：读取软件断点 |

## 关键流程

### 初始化与驱动装载

```text
UnrealDbgDll!Initialize(key)
    -> AIHelper!LoadNT("VT_Driver.sys", "VT_Driver")
    -> AIHelper!LoadNT("DbgkSysWin10.sys" 或 "DbgkSysWin11.sys", "UnrealDevice")
    -> CreateFile("\\.\UnrealDbg")
    -> IOCTL_LOAD_SYMBOLS_TABLE（加密密钥校验）
         -> 驱动解析 ntoskrnl / win32kbase / win32kfull 符号
         -> VMCALL_INIT_OFFSET 向 VT 宿主下发结构偏移
         -> DbgkInitialize + SetupEptHook
    -> IOCTL_LOAD_DEBUGGER_DATA（登记当前调试器进程）
```

### 调试器接管

```text
UnrealDbgDll!StartProcess(调试器路径, HookDLL目录)
    -> CreateProcess 拉起调试器
    -> 判断位数并选择 Hook64.dll / Hook.dll
    -> AIHelper!InjectDll 注入 Hook DLL
    -> Hook DllMain: SetupHook() 安装调试 API 钩子并连接设备
    -> IOCTL_LOAD_DEBUGGER_DATA 把调试器 PID 登记到驱动
    -> 驱动解析调试事件并重定向调试对象、隐藏断点
```

### 断点与内存监视

```text
设置软件断点（IOCTL_SET_SOFTWARE_BREAKPOINT）
    -> 驱动保存原始字节并写入 0xCC
    -> 通过 VMCALL_HIDE_SOFTWARE_BREAKPOINT 让目标视角仍看到原始字节
    -> 命中后由 EPT / VM-exit 上报，VMCALL_GET_BREAKPOINT 取回事件

EPT 内存断点（VMCALL_WATCH_READS / WRITES / EXECUTES）
    -> EPT 权限收紧产生 VM-exit
    -> 宿主记录命中信息并恢复执行
    -> VMCALL_WATCH_DELETE 删除监视
```

## 构建

### 环境要求

- Visual Studio 2022（工具集 `v143`），安装 C++ 桌面开发与 MASM；
- Windows Driver Kit：`WindowsTargetPlatformVersion` 为 `10.0.26100.0`（三个驱动工程）；
- 可选：VMProtect SDK，头文件与静态库已放在 `Common/VMProtect/`；
- 运行依赖：`AIHelper.dll`、`VMProtectSDK64.dll`（见 `_deps/`）。

### 编译命令

```powershell
msbuild UnrealDbg.sln /m /p:Configuration=Release /p:Platform=x64
```

注意：原始工程中 `UnrealDbgDll` 只有 `Release|x64` 配置为 `DynamicLibrary`，
其余配置为 `Application`，构建 DLL 时必须使用 `Release|x64`。
如果本机没有 WDK 10.0.26100.0，请先安装或修改驱动工程中的
`WindowsTargetPlatformVersion`。

另外，`Release|x64` 配置使用相对路径解析公共头文件，可以直接构建；
但工程中仍残留原作者机器的绝对路径：

- `VT_Driver.vcxproj`、`DbgkSysWin11.vcxproj`、`DbgkSysWin10.vcxproj` 的 `Debug|x64` 配置中，
  `IncludePath` 指向 `E:\Projects\VS\repos\UnrealDbg\Common`；
- `DbgkSysWin10.vcxproj` 的 `Release|x64` 还带有
  `E:\repos\调试器\UnrealDbg(gui)\Common\Ring0` 的附加包含目录与库目录；
- `DbgkSysWin10.vcxproj`、`DbgkSysWin11.vcxproj` 中存在指向
  `E:\repos\...\Common\Shared\` 的 `ClInclude` 条目。

这些路径不影响 `Release|x64` 构建（`Release` 配置已经使用 `..\Common`、
`$(SolutionDir)UnrealDbg\Common` 等相对路径），但切换到 `Debug|x64` 或整理工程时，
应把它们改成仓库内的相对路径。

### 产物对应关系

| 工程 | 配置 | 产物 |
|---|---|---|
| `VT_Driver` | Release x64 | `VT_Driver.sys` |
| `DbgkSysWin10` | Release x64 | `DbgkSysWin10.sys` |
| `DbgkSysWin11` | Release x64 | `DbgkSysWin11.sys` |
| `UnrealDbgDll` | Release x64 | `UnrealDbgDll.dll` |
| `Hook` | Release x64 / Win32 | `Hook64.dll` / `Hook.dll` |
| `Loader` | Release x64 | `VTDebugger.exe` |

## 运行与部署

典型部署文件：

```text
VTDebugger.exe          # Loader，首次运行可下载符号
UnrealDbgDll.dll        # Ring3 调度层
AIHelper.dll            # 驱动装载 / DLL 注入
VMProtectSDK64.dll      # VMProtect 运行库
VT_Driver.sys           # VT-x Hypervisor
DbgkSysWin10.sys        # 或 DbgkSysWin11.sys，按系统版本二选一
Hook.dll / Hook64.dll   # 注入调试器进程
C:\Symbols\             # ntoskrnl / win32kbase / win32kfull 符号
```

运行条件：

- Intel 处理器且 BIOS/UEFI 开启 VT-x（需要 EPT 支持）；
- Windows 10 / Windows 11 x64，管理员权限运行；
- 如果系统启用了 Hyper-V、VBS / 内存完整性等占用 VT-x 的功能，需要先在实验环境中关闭，
  否则 VT-x 无法再次被本驱动接管；
- 驱动需要数字签名；实验环境需开启测试签名模式或关闭驱动签名强制（DSE）；
- 符号文件需与当前系统版本匹配，否则驱动解析内部符号会失败。

发布提示：仓库的 `.gitignore` 会忽略 `*.dll`、`*.sys`、`*.exe`，并且整个 `_deps/`
（`AIHelper.dll`、`VMProtectSDK64.dll`、`UnrealDbg.aes` 等运行时依赖与发布产物）
都不会进入源码提交；部署或发布时需要把这些文件单独放到运行目录，或作为 Release 附件分发。

<a name="hybrid-ept"></a>

## 已知问题：Intel Core Ultra 大小核（P/E）的 EPT 适配

在 Intel Core Ultra（Meteor Lake / Arrow Lake / Lunar Lake 等）以及其它 P 核 + E 核混合架构上，
当前代码的 EPT 初始化还没有按逻辑处理器做能力适配。可能的表现：

- 驱动只在部分核心完成 VMXON / VMLAUNCH，其余核心初始化失败；
- 第一次 EPT 钩子或内存监视触发时出现虚拟机进入（VM-entry）失败或 EPT 配置错误（EPT misconfiguration）；
- 在不支持 INVEPT 的核心上，VMX 根模式（root mode）下执行 INVEPT 触发 #UD；未捕获时表现为
  `KMODE_EXCEPTION_NOT_HANDLED` 蓝屏。

### 代码层面的根因

| 位置 | 当前实现 | 异构核风险 |
|---|---|---|
| `VT_Driver/vmm.cpp: allocate_vmm_context()` | `ept::build_mtrr_map()`、`init_vcpu()`、`ept::initialize()` 都在驱动加载时所在的单个逻辑处理器上执行；此时还没有进入按核切换亲和性的循环 | 所有 vCPU 的 EPT 页表、`EPTP`、MTRR 缓存类型都来自同一个核的能力与配置 |
| `VT_Driver/EPT.cpp: initialize()` | 固定 `ept_pointer->memory_type = MEMORY_TYPE_WRITE_BACK`、`page_walk_length = 3`（4 级页遍历）；`create_ept_page_table()` 把所有 PDE 设为 2MB 大页 | 没有读取 `IA32_VMX_EPT_VPID_CAP` 的位 14（WB，写回）、位 6（4 级页遍历）、位 16（2MB 大页）、位 20 / 25 / 26（INVEPT）等能力位 |
| `VT_Driver/invalid_ept.cpp`、`vmexit_handler.cpp`、`EPT.cpp` | 无条件调用 `invept_all_contexts_func()` / `invept_single_context_func()`；`Globals.cpp: enter_vmx_operation()` 在 VMXON 成功后也会立即执行一次 INVEPT | 当前核不支持 INVEPT 时，VMX 根模式（root mode）下执行 INVEPT 会产生 #UD；内核未捕获时直接蓝屏 |
| `VT_Driver/Globals.cpp: enter_vmx_operation()` / `load_vmcs_pointer()` | 每次都在目标核上重新读取 `IA32_VMX_BASIC`，写入 VMXON / VMCS 的修订标识（revision ID） | 这部分已经是每核正确的，但 EPT 部分没有同样的处理 |
| `VT_Driver/vmcs.cpp: fill_vmcs()` / `ajdust_controls()` | 在目标核上读取 `IA32_VMX_*` 控制 MSR 并裁剪 VMCS 控制位 | VMCS 控制位会按核适配；但 `EPT_POINTER` 指向的 EPT 页表仍是单核构建的 |
| `VT_Driver/vmm.cpp: vmm_init()` | `KeQueryActiveProcessorCount(NULL)` + `1ull << iter` | 只覆盖当前处理器组；>64 逻辑处理器或跨处理器组的机型需要改用 `GROUP_AFFINITY` |

补充：`IA32_VMX_BASIC`、`IA32_VMX_EPT_VPID_CAP` 等 VMX 能力 MSR 都是每逻辑处理器 MSR。
Intel SDM 第 3C 卷的 VMX 能力报告（VMX Capability Reporting）要求软件在将要运行 VMX 的逻辑处理器上读取这些值；
P 核与 E 核属于不同微架构，报告的能力位可能不同，具体以实机 `rdmsr` 结果为准。

### 需要调整的 EPT 点（当前未实现）

1. 把 EPT / MTRR 的能力探测和页表构建移进每核初始化路径（`init_logical_processor()` 内、切换到目标核之后），
   或至少在每个核上重新读取并校验能力。
2. 每个核读取 `IA32_VMX_EPT_VPID_CAP`，至少处理：位 20（INVEPT）、位 25 / 26（单上下文 / 全上下文 INVEPT，single/all-context）、
   位 6（4 级页遍历）、位 14（WB，写回）、位 8（UC，不可缓存）、位 16（2MB 大页）、位 21（A/D，访问/脏位）。
3. 选择“所有核都支持”的公共 EPT 配置：仅在所有核支持 2MB 时使用大页；仅在所有核支持写回（WB）时使用 WB；
   `page_walk_length` 取公共支持值；否则退回 4KB 页 / 不可缓存（UC），或拒绝加载并输出明确的错误信息。
4. 给 INVEPT 增加能力检查；不支持 INVEPT 的核改用其它 TLB 失效路径，或直接返回不支持并报错，
   不要在 VMX 根模式（root mode）下无条件执行。
5. MTRR 缓存类型表改为每核读取 / 校验，至少校验 P 核与 E 核得到的 MTRR 配置是否一致。
6. 处理器遍历改为处理器组感知（`KeQueryActiveProcessorCountEx(ALL_PROCESSOR_GROUPS)` + `GROUP_AFFINITY`），
   并单独记录每个核的 VMXON / VMLAUNCH 结果；初始化中途失败时用 `hvgt::vmoff()` 做全核回滚
   （当前失败路径只调用 `hv::disable_vmx_operation()`，只影响当前核）。

### 实机验证建议

- 分别把线程固定到 P 核和 E 核，读取 `IA32_VMX_BASIC`、`IA32_VMX_EPT_VPID_CAP`，对比 VMCS 修订标识（revision ID）、
  INVEPT、2MB、WB 等能力位；
- 驱动加载失败时记录 `VM_INSTRUCTION_ERROR`（若出现 VMCS 修订标识不匹配，常见为错误码 12，
  具体以实测为准）和蓝屏（bugcheck）参数；
- 用 `KeSetSystemAffinityThreadEx()` 在单 P 核 / 单 E 核上分别加载驱动，确认问题是否与核类型相关。

## 已知限制

- 本仓库为原始代码整理版，未在仓库环境中完成构建和真机验证；
- 与系统版本强绑定：Win10 驱动以 19041.4291 内核头为基线，通过符号加特征码定位内部结构，
  系统更新后可能失效；
- 仅支持 Intel VT-x / EPT，不包含 AMD-V 实现；
- Intel Core Ultra 等 P/E 异构核平台的 EPT 尚未按核适配，详见上一节；当前代码未读取
  `IA32_VMX_EPT_VPID_CAP`。
- `Initialize` 的加密 KEY 与密钥校验值均为固定值，修改或更换后需要同步用户态与
  DbgkSys 驱动两侧；
- `SetupHook_DbgkCreateThread_CMP_Debugport` 当前为注释状态；
- `IOCTL_LOAD_DEBUGGER_STATE`、`IOCTL_LOAD_PROTECT_OBJ_DATA` 当前为预留分支；
- `UnInitialize` 中先卸载的服务名为 `BACDevice`（历史遗留），实际装载的服务名是
  `UnrealDevice` 与 `VT_Driver`，清理时需要留意；
- 部分功能依赖 VMProtect 商业 SDK；
- 仓库同时包含 Delphi 前端和配套工具，本文未覆盖其编译与运行方式。

## 免责声明

本项目仅用于个人学习、Windows 内核与硬件虚拟化技术研究，以及获得明确授权的安全测试环境。
禁止将其用于违反当地法律法规、侵犯他人权益、绕过在线游戏反作弊或破坏计算机系统的用途。
使用者应自行承担使用本项目带来的风险与后果。

---

<a name="english"></a>

[简体中文](#chinese) | **English**

# UnrealVTDbg (English)

**An Intel VT-x / EPT based Windows kernel debugging support framework (original source code)**

Status: original source (not fully built or validated on real hardware in this repository) · Platform: Windows 10 / 11 x64 · Dependencies: Intel VT-x + EPT

> Known issue: on Intel Core Ultra and other P-core + E-core hybrid platforms, EPT must be adapted per
> logical processor during initialization; see [Known Issue: Intel Core Ultra Hybrid (P/E) Core EPT Adaptation](#hybrid-ept-en).

UnrealVTDbg (the project name inside the code is UnrealDbg) is a self-contained debugging framework built around an in-kernel VT-x hypervisor. By combining EPT hooks, stealth breakpoints, and control over Dbgk (the Windows kernel debugging subsystem), it attaches a conventional user-mode debugger to a protected process and helps it resist driver-side anti-debugging checks. The project was originally built for Windows 10 / 11 x64 and is tightly coupled to the kernel symbols, structure offsets, and byte signatures of specific OS versions.

> Scope: this document covers only the C++ / driver components: `VT_Driver`, `DbgkSysWin10`, `DbgkSysWin11`, `UnrealDbgDll`, `Hook`, `Loader`, and `Common`. The Delphi front end and companion tools (`UnrealDbg/`, `CardRegistration/`, `D-encryption/`, `SymbolTool/`) are out of scope.
>
> This repository is a cleaned-up copy of the original source; a successful build or runtime is not guaranteed. Read [Known Limitations](#known-limitations) and [Disclaimer](#disclaimer) before building or running it.

## Core Capabilities

- **VT-x hypervisor host**: VMXON / VMLAUNCH, VMCS management, VM-exit dispatch, and a VMCALL gateway, with per-logical-processor vCPU contexts.
- **EPT memory virtualization**: EPT page tables and shadow page management; EPT execution hooks, read/write/execute memory watches, fake-page memory reads, and EPT function hooks.
- **Stealth breakpoints**: hide the real bytes of software breakpoints (0xCC), read back hidden breakpoint contents, and protect debug registers (DRx), so the target process and anti-debug drivers cannot see breakpoint traces.
- **VMCALL control interface**: 21 function IDs covering VMXOFF, INVEPT, EPT hook/unhook, hiding the hypervisor's presence, hiding/reading software breakpoints, EPT read/write/execute watches, and VMCS state dumps.
- **Dbgk takeover**: resolves symbols for `ntoskrnl`, `win32kbase`, and `win32kfull`, installs EPT function hooks on internal Dbgk / Psp routines, and redirects debug objects and the debug event stream.
- **Debugger-side hook DLL**: injected into a third-party debugger process to hook `NtDebugActiveProcess`, `WaitForDebugEvent`, `ContinueDebugEvent`, `ReadProcessMemory`, `WriteProcessMemory`, and other debugging APIs.
- **Kernel-level breakpoint management**: sets and removes hardware breakpoints (DRx) and software breakpoints over IOCTL; breakpoint state can be hidden by the VT layer.
- **Symbol loader**: downloads PDB symbols for `ntoskrnl.exe`, `win32kbase.sys`, and `win32kfull.sys` into `C:\Symbols\` so the driver can resolve internal kernel symbols.
- **Driver loading wrapper**: `AIHelper.dll` provides driver installation/uninstallation and DLL injection (`LoadNT` / `UnloadNT` / `InjectDll` / `outDebug`).

## System Architecture

```text
+---------------------------------------------------------------+
|            Delphi front end / external host (original, n/a)    |
+-------------------------------+-------------------------------+
                                | exported API calls
+-------------------------------v-------------------------------+
| UnrealDbgDll.dll (Ring3 dispatcher)                           |
|   Initialize / StartProcess / GetFileVersion                  |
|   TL_BlockGameResumeThread                                    |
+---------------+----------------------------+------------------+
                |                            |
                | AIHelper!LoadNT            | CreateProcess + InjectDll
                v                            v
+-----------------------------+   +------------------------------+
| VT_Driver.sys (hypervisor)  |   | Hook.dll / Hook64.dll         |
|   VMXON / VMCS / EPT        |<--|  debug API hooks + VMCALL     |
|   VM-exit / VMCALL dispatch |   |  Detours + EPT hooks          |
+--------------+--------------+   +---------------+--------------+
               ^                                  |
               | VMCALL (DPC broadcast to all LP) | IOCTL \\.\UnrealDbg
+--------------+----------------------------------v--------------+
| DbgkSysWin10.sys / DbgkSysWin11.sys (support drivers)          |
|   symbols / Dbgk hooks / breakpoints / key check / dispatch     |
+----------------------------------------------------------------+
```

Layer responsibilities:

| Layer | Component | Ring | Responsibility |
|---|---|---|---|
| Hypervisor | `VT_Driver.sys` | Ring 0 | Runs the original OS under Intel VT-x; provides EPT hooks and VMCALL services |
| Debug support driver | `DbgkSysWin10/11.sys` | Ring 0 | Resolves symbols, takes over Dbgk, manages breakpoints, encrypts/decrypts IOCTL data |
| Dispatcher | `UnrealDbgDll.dll` | Ring 3 | Loads drivers, sends keys and debugger data, starts and injects the debugger |
| Debugger side | `Hook.dll` / `Hook64.dll` | Target debugger process | Hooks debugging APIs and coordinates with the drivers via IOCTL / VMCALL |
| Helper | `Loader` (`VTDebugger.exe`) | Ring 3 | Symbol download and management |
| Shared library | `Common/` | All layers | Detours, cryptography, logging, PDB parsing, shared structures |

## Repository Layout

```text
.
├─ UnrealDbg.sln              # VS solution (6 C++ projects)
├─ VT_Driver/                 # VT-x hypervisor host driver (C++ / ASM, WDM)
├─ DbgkSysWin10/              # Windows 10 debug support driver (WDM)
├─ DbgkSysWin11/              # Windows 11 debug support driver (WDM)
├─ UnrealDbgDll/              # Ring3 dispatcher DLL (exported API)
├─ Hook/                      # Hook DLL injected into the debugger (x86 / x64)
├─ Loader/                    # symbol loader, output: VTDebugger.exe
├─ Common/                    # shared libraries (Detours, crypto, logging, PDB parsing, ...)
├─ _deps/                     # prebuilt runtime dependencies (AIHelper, VMProtect runtime, ...)
├─ Common/VT_Driver/          # shared VT/EPT support code for drivers
├─ UnrealDbg/                 # original Delphi front end (out of scope)
├─ CardRegistration/          # license registration tool (Delphi, out of scope)
├─ D-encryption/              # encryption tool (Delphi, out of scope)
├─ D-encryptiondll/           # encryption DLL source (companion tool)
└─ SymbolTool/                # symbol tool (Delphi, out of scope)
```

## Module Details

### 1. VT_Driver (Hypervisor Host)

The execution order of `DriverEntry` (`VT_Driver/Driver.cpp`):

1. Call `InitNtoskrnlSymbolsTable()` to resolve `ntoskrnl` symbols.
2. Call `hv::virtualization_support()` to check whether the CPU supports VMX operation.
3. Call `hv::InitGlobalVariables()` to initialize global state, then `vmm_init()` to start VT on each logical processor.
4. On failure, call `hv::disable_vmx_operation()` and release the vCPU contexts.

`Unload` first removes all EPT hooks (`hvgt::ept_unhook()`), then calls `hvgt::vmoff()` on every logical processor, and finally disables VMX and frees resources.

Key files:

| File | Description |
|---|---|
| `Driver.cpp` | Driver entry/exit, VT and symbol table initialization |
| `vmm.cpp` / `vmcs.cpp` | vCPU management, VMCS field access and initialization |
| `vmexit_handler.cpp` | VM-exit dispatch: instruction adjustment, exception injection, EPT handling, etc. |
| `vmcall_handler.cpp` | VMCALL function ID dispatch and argument handling |
| `EPT.cpp` | EPT page tables, hook pages, memory watches, and fake-page implementation |
| `hypervisor_gateway.cpp` | Host-side wrappers: EPT hooks, INVEPT, VMXOFF, hiding the hypervisor, etc. |
| `ASM/*.asm` | VMX instruction wrappers, VM-exit entry, interrupt handlers, and the LDE64 length disassembler |
| `crx.h` / `drx.h` / `msr.h` / `mtrr.h` | Control registers, debug registers, MSR, and MTRR support |

Note: the current implementation targets Intel VT-x only (`vmx.h` / `VMX_*`); there is no AMD-V path.

### 2. DbgkSysWin10 / DbgkSysWin11 (Debug Support Drivers)

The two directories are structurally almost identical (62 source files each); the differences are the OS-version-specific symbols, structure offsets, and byte signatures. `UnrealDbgDll` selects the driver by build number: `>= 22000` uses the Win11 build, otherwise it uses the Win10 build.

Device interface (identical in both versions):

- Device name: `\Device\DbgkSysDevice`
- Symbolic link: `\??\UnrealDbg`, user-mode path `\\.\UnrealDbg`
- Buffering: `DO_BUFFERED_IO`; `IRP_MJ_DEVICE_CONTROL` is handled by `HandlerDispatchRoutin`

Main mechanisms:

1. **Encrypted IOCTL payloads**: user mode encrypts plaintext (`USER_DATA`) with Blowfish; the driver decrypts it and validates length/content. After a successful fixed-key check, it writes back the magic value `1998`.
2. **Symbol resolution**: after key validation it calls `InitNtoskrnlSymbolsTable()`, `InitWin32kbaseSymbolsTable()`, and `InitWin32kfullSymbolsTable()` to resolve internal kernel symbols such as `Sys_DbgkCreateThread`, `Sys_PspExitThread`, `Sys_DbgkExitThread`, `DbgkDebugObjectType`, and `Sys_MmProtectVirtualMemory`.
3. **Offset delivery**: `DispatchOffsetToHost()` sends kernel structure offsets (for example `ETHREAD.Cid`) to the VT host through `VMCALL_INIT_OFFSET`.
4. **Dbgk initialization**: calls `DbgkInitialize()` and `SetupEptHook()` to install the takeover logic.
5. **EPT function hooks**: `hvgt::hook_function()` (which broadcasts `VMCALL_EPT_CC_HOOK`) hooks kernel routines such as `NtTerminateProcess`, `PspExitThread`, and `PspCreateThread`. In the original source, `SetupHook_DbgkCreateThread_CMP_Debugport()` is commented out while `SetupHook_PspExitThread_CMP_Debugport()` is active.
6. **Protection and countermeasure modules**: `Protect/` contains process protection, thread DRx protection, and `BypassFindWnd` (window-finding bypass) logic; `DbgkApi/` provides debug object and Dbgk message handling.

`ntos/inc/10.0.19041.4291/` indicates that the Win10 build is based on the 19041.4291 kernel headers; other internal structures are located through symbols plus byte signatures. OS updates that change symbols or code patterns will break initialization.

### 3. UnrealDbgDll (Ring3 Dispatcher)

Exported API (`UnrealDbgDll/dllexport.def`):

| Export | Description |
|---|---|
| `Initialize(key)` | Loads drivers, sends the key, initializes the symbol table, and registers the current debugger process |
| `StartProcess(szExe, sPath)` | Creates the target debugger process, injects `Hook/Hook64`, and reports debugger information |
| `GetFileVersion` | Reads file version information |
| `TL_BlockGameResumeThread(pid)` | Blocks the target process's resume thread through IOCTL |

Actual flow of `Initialize`:

1. `_Initialize(module directory)` uses `AIHelper.dll!LoadNT` to load `VT_Driver.sys` (service name `VT_Driver`) and then `DbgkSysWin10.sys` / `DbgkSysWin11.sys` (service name `UnrealDevice`), and then opens the `\\.\UnrealDbg` device.
2. `LoadSymbolsTable(key)` sends the encrypted `RING3_VERIFY` structure through `IOCTL_LOAD_SYMBOLS_TABLE` and waits for the success value `1998`.
3. On success, `SendDebuggerDataToDriver(GetCurrentProcessId())` registers the current debugger process through `IOCTL_LOAD_DEBUGGER_DATA`.
4. The key validation logic is wrapped in `VMProtectBeginVirtualization` / `VMProtectEnd` and links against `Common/VMProtect/VMProtectSDK64.lib`.

Actual flow of `StartProcess`:

1. `CreateProcess` starts the specified debugger executable.
2. `IsWow64Process` determines the bitness and selects `Hook64.dll` or `Hook.dll`.
3. `AIHelper.dll!InjectDll` injects the hook DLL.
4. `SendDebuggerDataToDriver(pid)` registers the debugger process with the driver.

Note: `DispatchSymbol()` is currently an empty function (the old reporting path is commented out); the active path is key validation plus debugger registration inside `Initialize`.

### 4. Hook / Hook64 (Debugger-side Hook DLL)

After being injected into the debugger process, `DllMain` runs the following on `DLL_PROCESS_ATTACH`:

```text
InitGlobalVariables() -> InitFunction() -> SetupHook()
```

After connecting to the `\\.\UnrealDbg` device, `SetupHook()` installs the following Detours hooks (the hook DLL and the driver cooperate; some capabilities are ultimately implemented with EPT):

| Category | Hooked functions |
|---|---|
| Debug attachment | `NtDebugActiveProcess`, `DbgUiIssueRemoteBreakin` |
| Debug events | `WaitForDebugEvent`, `ContinueDebugEvent` |
| Output channel | `OutputDebugStringA`, `OutputDebugStringW` |
| Thread context | `GetThreadContext`, `SetThreadContext` |
| Memory operations | `ReadProcessMemory`, `WriteProcessMemory`, `VirtualProtectEx` |

`UnHook()` also contains unhook logic for `DebugActiveProcess`, `DbgUiDebugActiveProcess`, and `NtCreateUserProcess`, which are reserved or legacy paths.

Other files:

| Path | Description |
|---|---|
| `vmx/vmx.cpp` | User-mode VMCALL wrappers: `__vm_call`, `vmcall`, `vmcall2`, `current_vmcall` |
| `DebugEvent/` | Debug event takeover and conversion |
| `Inject/ApcInject/` | APC injection implementation |
| `Inject/ShellCode/` | Shellcode injection implementation |
| `Channels/` | Data marshalling for driver communication |
| `HookCallSet/functionSet.cpp` | Implementation of the individual hook functions |

### 5. Loader (VTDebugger.exe)

`Loader/WinMain.cpp` is a symbol downloader with a progress display:

- on startup it asks whether to skip the symbol download;
- it processes `ntoskrnl.exe`, `win32kbase.sys`, and `win32kfull.sys` in order;
- it reports progress through the URLMON `IBindStatusCallback` interface and downloads the PDB files to `C:\Symbols\`;
- the downloaded symbols are consumed by the `DbgkSysWin10/11` drivers to resolve internal kernel symbols.

The project `TargetName` is `VTDebugger`.

### 6. Common (Shared Libraries)

| Directory | Description |
|---|---|
| `Shared/` | Communication contracts such as `SharedStruct.h` and `IOCTLs.h` |
| `Detours/` | Detours headers, x86/x64 static libraries, and hook wrappers |
| `Encrypt/Blowfish/` | Blowfish encryption/decryption implementation |
| `Hash/` | MD5 and CRC32 |
| `FileSystem/` | File and path utilities |
| `Logger/` | Logging component |
| `IPC/SharedMemory/` | Shared-memory communication |
| `Ring0/SymbolicAccess/` | Driver-side PDB parsing, symbol extraction, Phnt headers, and ia32-doc definitions |
| `VMProtect/` | VMProtect SDK headers and x86/x64 static libraries |
| `VT_Driver/` | Shared VT/EPT support code for drivers |

## Interfaces

### VMCALL (Guest to Hypervisor)

Function IDs are defined in `VT_Driver/vmcall_reason.h` and dispatched by the VT host in `vmcall_handler.cpp`:

| ID | Name | Purpose |
|---|---|---|
| 0 | `VMCALL_TEST` | Test the VMCALL channel |
| 1 | `VMCALL_VMXOFF` | Turn off VMX |
| 2 | `VMCALL_EPT_CC_HOOK` | EPT-based 0xCC execution hook |
| 3 | `VMCALL_EPT_INT1_HOOK` | EPT-based INT1 hook |
| 4 | `VMCALL_EPT_RIP_HOOK` | EPT-based RIP redirect hook |
| 5 | `VMCALL_EPT_HOOK_FUNCTION` | Function-level EPT hook |
| 6 | `VMCALL_EPT_UNHOOK_FUNCTION` | Remove a function-level EPT hook |
| 7 | `VMCALL_INVEPT_CONTEXT` | Invalidate the EPT TLB context |
| 8 | `VMCALL_DUMP_POOL_MANAGER` | Dump pool manager information |
| 9 | `VMCALL_DUMP_VMCS_STATE` | Dump the VMCS state |
| 10 | `VMCALL_HIDE_HV_PRESENCE` | Hide the hypervisor's presence (CPUID) |
| 11 | `VMCALL_UNHIDE_HV_PRESENCE` | Make the hypervisor visible again |
| 12 | `VMCALL_HIDE_SOFTWARE_BREAKPOINT` | Hide a software breakpoint |
| 13 | `VMCALL_READ_SOFTWARE_BREAKPOINT` | Read back a hidden software breakpoint |
| 14 | `VMCALL_READ_EPT_FAKE_PAGE_MEMORY` | Read original memory from an EPT fake page |
| 15 | `VMCALL_WATCH_WRITES` | EPT write watch |
| 16 | `VMCALL_WATCH_READS` | EPT read watch |
| 17 | `VMCALL_WATCH_EXECUTES` | EPT execute watch |
| 18 | `VMCALL_WATCH_DELETE` | Delete a watch entry |
| 19 | `VMCALL_GET_BREAKPOINT` | Retrieve information about a hit breakpoint |
| 20 | `VMCALL_INIT_OFFSET` | Deliver structure offsets to the host |

### IOCTL (Ring3 to DbgkSys)

Definitions live in `Common/Shared/IOCTLs.h`, with an identical copy under `UnrealDbg/Common/Shared/`. All of them use `FILE_DEVICE_UNKNOWN` + `METHOD_BUFFERED` + `FILE_ANY_ACCESS`. The `0x800` to `0x80B` values are CTL_CODE function numbers; the actual control codes also include device-type bits (for example, function `0x800` produces control code `0x222000`):

| Function | Name | Status |
|---|---|---|
| `0x800` | `IOCTL_LOAD_SYMBOLS_TABLE` | Implemented: decrypt, key check, symbol resolution, and initialization |
| `0x801` | `IOCTL_LOAD_DEBUGGER_STATE` | Reserved; the current branch is empty |
| `0x802` | `IOCTL_LOAD_PROTECT_OBJ_DATA` | Reserved; the current branch is empty |
| `0x803` | `IOCTL_LOAD_DEBUGGER_DATA` | Implemented: register the debugger process |
| `0x804` | `IOCTL_CREATE_REMOTE_THREAD` | Implemented: create a remote thread from the kernel |
| `0x805` | `IOCTL_SET_HARDWARE_BREAKPOINT` | Implemented: set a hardware breakpoint |
| `0x806` | `IOCTL_GET_PROCESS_INFO` | Implemented: query target process information |
| `0x807` | `IOCTL_TL_BLOCK_RESUME_THREAD` | Implemented: block the game's resume thread |
| `0x808` | `IOCTL_DEL_HARDWARE_BREAKPOINT` | Implemented: remove a hardware breakpoint |
| `0x809` | `IOCTL_SET_SOFTWARE_BREAKPOINT` | Implemented: set a software breakpoint |
| `0x80A` | `IOCTL_DEL_SOFTWARE_BREAKPOINT` | Implemented: remove a software breakpoint |
| `0x80B` | `IOCTL_READ_SOFTWARE_BREAKPOINT` | Implemented: read a software breakpoint |

## Key Flows

### Initialization and Driver Loading

```text
UnrealDbgDll!Initialize(key)
    -> AIHelper!LoadNT("VT_Driver.sys", "VT_Driver")
    -> AIHelper!LoadNT("DbgkSysWin10.sys" or "DbgkSysWin11.sys", "UnrealDevice")
    -> CreateFile("\\.\UnrealDbg")
    -> IOCTL_LOAD_SYMBOLS_TABLE (encrypted key check)
         -> driver resolves ntoskrnl / win32kbase / win32kfull symbols
         -> VMCALL_INIT_OFFSET sends structure offsets to the VT host
         -> DbgkInitialize + SetupEptHook
    -> IOCTL_LOAD_DEBUGGER_DATA (register the current debugger process)
```

### Debugger Takeover

```text
UnrealDbgDll!StartProcess(debugger path, HookDLL directory)
    -> CreateProcess starts the debugger
    -> determine bitness and select Hook64.dll / Hook.dll
    -> AIHelper!InjectDll injects the hook DLL
    -> Hook DllMain: SetupHook() installs debug API hooks and connects to the device
    -> IOCTL_LOAD_DEBUGGER_DATA registers the debugger PID with the driver
    -> the driver parses debug events, redirects debug objects, and hides breakpoints
```

### Breakpoints and Memory Watches

```text
Set a software breakpoint (IOCTL_SET_SOFTWARE_BREAKPOINT)
    -> the driver saves the original bytes and writes 0xCC
    -> VMCALL_HIDE_SOFTWARE_BREAKPOINT keeps the original bytes visible to the target
    -> on hit, EPT / VM-exit reports the event and VMCALL_GET_BREAKPOINT retrieves it

EPT memory breakpoints (VMCALL_WATCH_READS / WRITES / EXECUTES)
    -> tightening EPT permissions triggers a VM-exit
    -> the host records the hit information and resumes execution
    -> VMCALL_WATCH_DELETE removes the watch
```

## Build

### Requirements

- Visual Studio 2022 (toolset `v143`) with Desktop C++ and MASM;
- Windows Driver Kit: `WindowsTargetPlatformVersion` is `10.0.26100.0` (three driver projects);
- Optional: VMProtect SDK; headers and static libraries are already under `Common/VMProtect/`;
- Runtime dependencies: `AIHelper.dll`, `VMProtectSDK64.dll` (see `_deps/`).

### Build Command

```powershell
msbuild UnrealDbg.sln /m /p:Configuration=Release /p:Platform=x64
```

Note: only `Release|x64` of `UnrealDbgDll` is configured as `DynamicLibrary`; the other configurations are `Application`, so use `Release|x64` to build the DLL. If WDK 10.0.26100.0 is not installed, install it or adjust `WindowsTargetPlatformVersion` in the driver projects.

In addition, the `Release|x64` configurations use relative paths to resolve shared headers and can be built directly, but some absolute paths from the original author's machine remain:

- the `Debug|x64` configurations of `VT_Driver.vcxproj`, `DbgkSysWin11.vcxproj`, and `DbgkSysWin10.vcxproj` point `IncludePath` to `E:\Projects\VS\repos\UnrealDbg\Common`;
- `Release|x64` of `DbgkSysWin10.vcxproj` still carries `E:\repos\调试器\UnrealDbg(gui)\Common\Ring0` as an additional include and library directory;
- `DbgkSysWin10.vcxproj` and `DbgkSysWin11.vcxproj` contain `ClInclude` entries pointing to `E:\repos\...\Common\Shared\`.

These paths do not affect `Release|x64` builds (the Release configurations already use relative paths such as `..\Common` and `$(SolutionDir)UnrealDbg\Common`), but they should be replaced with repository-relative paths when switching to `Debug|x64` or cleaning up the projects.

### Output Mapping

| Project | Configuration | Output |
|---|---|---|
| `VT_Driver` | Release x64 | `VT_Driver.sys` |
| `DbgkSysWin10` | Release x64 | `DbgkSysWin10.sys` |
| `DbgkSysWin11` | Release x64 | `DbgkSysWin11.sys` |
| `UnrealDbgDll` | Release x64 | `UnrealDbgDll.dll` |
| `Hook` | Release x64 / Win32 | `Hook64.dll` / `Hook.dll` |
| `Loader` | Release x64 | `VTDebugger.exe` |

## Deployment and Runtime

Typical deployment files:

```text
VTDebugger.exe          # Loader; downloads symbols on first run
UnrealDbgDll.dll        # Ring3 dispatcher
AIHelper.dll            # driver installation / DLL injection
VMProtectSDK64.dll      # VMProtect runtime
VT_Driver.sys           # VT-x hypervisor
DbgkSysWin10.sys        # or DbgkSysWin11.sys, depending on the OS version
Hook.dll / Hook64.dll   # injected into the debugger process
C:\Symbols\             # ntoskrnl / win32kbase / win32kfull symbols
```

Runtime conditions:

- an Intel CPU with VT-x enabled in BIOS/UEFI (EPT support is required);
- Windows 10 / 11 x64, running with administrator privileges;
- if Hyper-V, VBS / Memory Integrity, or another feature that occupies VT-x is enabled, it must be disabled in the lab environment first, otherwise the driver cannot take over VT-x;
- drivers must be digitally signed; a lab environment needs test-signing mode or Driver Signature Enforcement (DSE) disabled;
- the symbol files must match the current OS version, otherwise the driver cannot resolve internal symbols.

Publishing note: the repository `.gitignore` ignores `*.dll`, `*.sys`, and `*.exe`, and the entire `_deps/` directory (`AIHelper.dll`, `VMProtectSDK64.dll`, `UnrealDbg.aes`, and other runtime dependencies and release artifacts) is excluded from source commits. Deploy or release them separately in the runtime directory or as Release assets.

<a name="hybrid-ept-en"></a>

## Known Issue: Intel Core Ultra Hybrid (P/E) Core EPT Adaptation

On Intel Core Ultra (Meteor Lake / Arrow Lake / Lunar Lake, etc.) and other P-core + E-core hybrid platforms, the current EPT initialization is not adapted per logical processor. Possible symptoms:

- the driver completes VMXON / VMLAUNCH on only some cores while initialization fails on the others;
- the first EPT hook or memory watch triggers a VM-entry failure or an EPT misconfiguration;
- on a core that does not support INVEPT, executing INVEPT in root mode raises #UD, which appears as a `KMODE_EXCEPTION_NOT_HANDLED` bugcheck when unhandled.

### Root Causes in the Code

| Location | Current implementation | Risk on hybrid cores |
|---|---|---|
| `VT_Driver/vmm.cpp: allocate_vmm_context()` | `ept::build_mtrr_map()`, `init_vcpu()`, and `ept::initialize()` all run on the single logical processor that loaded the driver; the per-core affinity loop has not started yet | EPT page tables, `EPTP`, and MTRR memory types for every vCPU come from that one core's capabilities and configuration |
| `VT_Driver/EPT.cpp: initialize()` | Hard-codes `ept_pointer->memory_type = MEMORY_TYPE_WRITE_BACK` and `page_walk_length = 3` (4-level walk); `create_ept_page_table()` marks every PDE as a 2MB large page | Never reads the `IA32_VMX_EPT_VPID_CAP` bits 14 (WB), 6 (4-level), 16 (2MB), or 20 / 25 / 26 (INVEPT) |
| `VT_Driver/invalid_ept.cpp`, `vmexit_handler.cpp`, `EPT.cpp` | Calls `invept_all_contexts_func()` / `invept_single_context_func()` unconditionally; `Globals.cpp: enter_vmx_operation()` also executes INVEPT immediately after VMXON succeeds | If the current core does not support INVEPT, executing it in root mode raises #UD and bugchecks the kernel when unhandled |
| `VT_Driver/Globals.cpp: enter_vmx_operation()` / `load_vmcs_pointer()` | Re-reads `IA32_VMX_BASIC` on the target core and writes the VMXON / VMCS revision ID each time | This part is already per-core correct, but the EPT path has no equivalent handling |
| `VT_Driver/vmcs.cpp: fill_vmcs()` / `ajdust_controls()` | Reads the `IA32_VMX_*` control MSRs on the target core and clamps the VMCS control fields | VMCS controls are adapted per core, but `EPT_POINTER` still points at an EPT built for a single core |
| `VT_Driver/vmm.cpp: vmm_init()` | `KeQueryActiveProcessorCount(NULL)` + `1ull << iter` | Covers only the current processor group; machines with more than 64 logical processors or multiple groups need `GROUP_AFFINITY` |

Note: VMX capability MSRs such as `IA32_VMX_BASIC` and `IA32_VMX_EPT_VPID_CAP` are per-logical-processor MSRs. Intel SDM Vol. 3C (VMX Capability Reporting) requires software to read these values on the logical processor that will execute VMX; P-cores and E-cores are different microarchitectures and may report different capability bits. Always confirm with an actual `rdmsr` on the target machine.

### EPT Changes Required (not implemented yet)

1. Move EPT / MTRR capability discovery and page-table construction into the per-core initialization path (inside `init_logical_processor()`, after switching to the target core), or at least re-read and verify the capabilities on every core.
2. Read `IA32_VMX_EPT_VPID_CAP` per core and handle at least: bit 20 (INVEPT), bits 25 / 26 (single / all-context INVEPT), bit 6 (4-level walk), bit 14 (WB), bit 8 (UC), bit 16 (2MB), and bit 21 (A/D).
3. Choose a common EPT configuration supported by every core: use large pages only when all cores support 2MB; use WB only when all cores support it; take `page_walk_length` from the common capability set; otherwise fall back to 4KB pages / UC, or refuse to load with a clear error.
4. Guard INVEPT with a capability check; use another TLB invalidation path on cores that do not support it, or fail explicitly. Never execute it unconditionally in root mode.
5. Read and verify the MTRR cache-type table per core, or at least verify that P-cores and E-cores report the same MTRR configuration.
6. Make processor enumeration processor-group aware (`KeQueryActiveProcessorCountEx(ALL_PROCESSOR_GROUPS)` + `GROUP_AFFINITY`), record VMXON / VMLAUNCH results per core, and use `hvgt::vmoff()` for a full rollback when initialization fails halfway (the current failure path only calls `hv::disable_vmx_operation()`, which affects the current core only).

### Verification Suggestions

- Pin threads to a P-core and an E-core separately, read `IA32_VMX_BASIC` and `IA32_VMX_EPT_VPID_CAP`, and compare the VMCS revision ID, INVEPT, 2MB, and WB capability bits;
- capture `VM_INSTRUCTION_ERROR` and the bugcheck parameters when driver loading fails (a VMCS revision mismatch usually appears as VM-instruction error 12; confirm on the target machine);
- load the driver with `KeSetSystemAffinityThreadEx()` pinned to a single P-core and a single E-core to confirm whether the failure correlates with the core type.

## Known Limitations

- this repository is a cleaned-up copy of the original source and has not been fully built or validated on real hardware here;
- it is tightly coupled to OS versions: the Win10 driver is based on the 19041.4291 kernel headers and locates internal structures through symbols plus byte signatures, so OS updates may break it;
- Intel VT-x / EPT only; there is no AMD-V implementation;
- EPT is not yet adapted per logical processor on Intel Core Ultra (P/E hybrid) platforms;
  see the previous section. The current code never reads `IA32_VMX_EPT_VPID_CAP`.
- the encryption key and key check value used by `Initialize` are hard-coded; changing them requires updating both user mode and the DbgkSys driver;
- `SetupHook_DbgkCreateThread_CMP_Debugport` is currently commented out;
- `IOCTL_LOAD_DEBUGGER_STATE` and `IOCTL_LOAD_PROTECT_OBJ_DATA` are currently reserved stubs;
- `UnInitialize` unloads `BACDevice` first (a legacy service name), while the actual services are `UnrealDevice` and `VT_Driver`, so cleanup code should be aware of this;
- some functionality depends on the commercial VMProtect SDK;
- the repository also contains a Delphi front end and companion tools whose build and run instructions are not covered here.

## Disclaimer

This project is intended only for personal study, Windows kernel and hardware virtualization research, and explicitly authorized security testing environments. Do not use it to violate local laws, infringe on the rights of others, bypass online game anti-cheat systems, or damage computer systems. You are responsible for any risks and consequences of using this project.

---

[Back to top](#chinese)

[简体中文](README.md) | **English**

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

[Back to top](#unrealvtdbg-english) | [简体中文](README.md)

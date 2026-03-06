Injection Method: Manual Mapping
The project uses Manual Mapping, which means manually mapping the DLL's PE (Programmable Execution Environment) into memory within the target process and performing relocation, import table processing, etc., without using LoadLibrary. Therefore:
The module (InProcModuleList, etc.) will not be registered in the target process's PEB (Programmable Execution Environment).
Many PEB-only scans will not detect this DLL.
Suitable for injections without a module list.
The overall process is as follows:
In the target process, use VirtualAllocEx to allocate a block of read-write memory based on the DLL's SizeOfImage.Write the parsed DLL data (PE header + sections) into the memory using WriteProcessMemory.Allocate another block of memory in the target process, write a piece of shellcode, and also write a MANUAL_MAPPING_DATA structure (containing LoadLibraryA, GetProcAddress, RtlAddFunctionTable, DLL base address, etc.) into the target process.
Execute this shellcode in the target process using CreateRemoteThread.
The shellcode is executed within the target process as follows: Base Relocation
Import table resolution (filling the IAT with the passed LoadLibraryA / GetProcAddress)
TLS callback (if present)SEH table registration (RtlAddFunctionTable, optional)
Finally, the DLL's DllMain(DLL_PROCESS_ATTACH) is called.Therefore: Injection method = Manual Mapping + CreateRemoteThread execution of shellcode.Function Overview

1. Processes and Injection

Find the process by name (Process.cpp): Use `GetPIDByName` with `CreateToolhelp32Snapshot + Process32First/Next` to find the PID by name.

Open the target process (OpenProc): Use `OpenProcess` to obtain a handle with permissions including `PROCESS_CREATE_THREAD` | `PROCESS_VM_*`, for subsequent memory writing and remote thread creation.

Default target: `notepad.exe`; other processes can be specified using the command line using `-proc process_name`, for example: `injector.exe -proc target.exe`.

2. Payload Form (Embedded DLL)

The DLL is not loaded from disk, but is embedded in the injector during compilation: `payload_dll_data.cpp` contains a `g_embedded_payload_dll[]` byte array (MZ in the PE header), and `g_embedded_payload_dll_size` is its size.

During injection, this memory segment is directly assigned to `ManualMap(hProc, g_embedded_payload_dll, g_embedded_payload_dll_size)`, mapping this PE file in the target process.

In other words, the payload is an "embedded DLL + Manual Mapping".

3. Anti-detection/Stealth

String obfuscation (main.cpp): The template `XorStr` performs an XOR operation on the string at compile time and decrypts it at runtime, preventing plaintext errors such as "notepad.exe" and "Injection error".

Self-copying and deletion of the original file (SpawnRandomCopy):

First, it copies itself into an exe with a random name (e.g., xxxxxxxx.exe);

The `CreateProcess` command starts this copy, passing the parameters `--child` and the path to the original exe;

In the child process, it sleeps and then uses `DeleteFileA` to delete the original exe, then continues the injection.

The effect is: after execution, the original file is deleted, leaving only a randomly named copy running.

Random console title: SetConsoleTitleA(RandomString(12)), reducing fixed features.

4. Manual Mapping Internal Details (injection.cpp)

Verifies DOS header e_magic == 0x5A4D.

Allocates and writes the header and all sections in the target process according to the PE's SizeOfImage; sets VirtualProtectEx by section attributes (read-only/writable/executable).

Uses RELOC_FLAG64 for DIR64 relocation (64-bit) in the shellcode.

Supports import by sequence number (IMAGE_SNAP_BY_ORDINAL) and import by name.

Supports TLS callbacks and SEH (RtlAddFunctionTable); if SEH registration fails, hMod is set to 0x505050 to indicate incomplete exception handling.

After injection, the memory containing the shellcode and MANUAL_MAPPING_DATA is erased (written to 0 and protected/released), reducing residual signatures.

Polling `ReadProcessMemory` checks if `MANUAL_MAPPING_DATA.hMod` has been filled with shellcode to confirm that `DllMain` has been executed; a value of 0x404040 indicates an error within the shellcode.

5. Execution Behavior
After successful injection, it will print "[+] Injected!", then suspend indefinitely using `while (true) Sleep(1000)` without exiting (possibly to maintain the process or facilitate debugging).

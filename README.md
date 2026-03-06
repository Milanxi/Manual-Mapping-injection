功能概览
1. 进程与注入
按进程名找进程（Process.cpp）：GetPIDByName 用 CreateToolhelp32Snapshot + Process32First/Next 按名称查 PID。
打开目标进程（OpenProc）：用 OpenProcess 拿到句柄，权限包含 PROCESS_CREATE_THREAD | PROCESS_VM_*，供后续写内存和创建远程线程。
默认目标：notepad.exe；命令行可用 -proc 进程名 指定其它进程，例如：injector.exe -proc target.exe。
2. Payload 形式（内嵌 DLL）
DLL 不是从磁盘加载，而是 编译时内嵌 在注入器里：payload_dll_data.cpp 里是 g_embedded_payload_dll[] 字节数组（PE 头为 MZ），g_embedded_payload_dll_size 为其大小。
注入时直接把这段内存交给 ManualMap(hProc, g_embedded_payload_dll, g_embedded_payload_dll_size)，在目标进程里映射这份 PE。
也就是说：Payload 是“内嵌 DLL + Manual Mapping”。
3. 反检测 / 隐蔽
字符串混淆（main.cpp）：用模板 XorStr 在编译期对字符串做 XOR，运行时再解密，避免明文出现 "notepad.exe"、"Injection error" 等。
自拷贝并删除原文件（SpawnRandomCopy）：
先把自身复制成一个 随机名 exe（如 xxxxxxxx.exe）；
用 CreateProcess 启动这个副本，并传参 --child 和原 exe 路径；
子进程里 Sleep 后 DeleteFileA 删掉原 exe，然后继续执行注入。
效果是：运行后原文件被删，只留下一个随机名的副本在跑。
随机控制台标题：SetConsoleTitleA(RandomString(12))，减少固定特征。
4. Manual Mapping 内部细节（injection.cpp）
校验 DOS 头 e_magic == 0x5A4D。
按 PE 的 SizeOfImage 在目标进程分配并写入头 + 所有节；按节属性设置 VirtualProtectEx（只读/可写/可执行）。
Shellcode 里用 RELOC_FLAG64 做 DIR64 重定位（64 位）。
支持 按序号导入（IMAGE_SNAP_BY_ORDINAL）和按名称导入。
支持 TLS 回调 和 SEH（RtlAddFunctionTable）；若 SEH 注册失败，会把 hMod 设为 0x505050 表示异常处理不完整。
注入完成后会 擦掉 Shellcode 和 MANUAL_MAPPING_DATA 所在内存（写 0 并改保护/释放），减少残留特征。
通过轮询 ReadProcessMemory 看 MANUAL_MAPPING_DATA.hMod 是否被 Shellcode 填上，以确认 DllMain 已执行；若为 0x404040 表示 Shellcode 内出错。
5. 运行行为
注入成功后会打印 [+] Injected!，然后 while (true) Sleep(1000) 一直挂起不退出（可能是为了保持进程或方便调试）。

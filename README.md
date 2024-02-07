# Find specific memory allocations 

This repository contains a script to be used in by the `perf` tool, to dump callstacks for syscall calls related to a certain memory address. This is usefull if you want to find the causer that allocated specific memory pages.

## 1. Create an `perf trace` of a specific process

This example shows how to collect perf syscall related trace events for a given process id for certain time:

```zsh
TARGET_PID=12345
perf trace record --call-graph fp -p $TARGET_PID -e 'syscalls:*_mmap,syscalls:*_munmap,syscalls:*_brk,syscalls:*_mremap' -o "~/traces/mytrace.data --sleep 60m
```

## 2. Configure the script `find-specific-memory-allocation.py`
As first step, I would run the script dumping information for all memory pages to look at the callstacks and see if the debug symbols are correctly loaded (see chapter 4.). To achieve this open the script and set `memory_pages_to_consider = None`

If the stack traces are looking correct, change the script to only look for certain memory pages:

```python
memory_pages_to_consider = { 0x78e8c18e7000, 0x78e8fcafe000 }
```

## 3. Run perf with the script 

```zsh
perf script -i ~/traces/mytrace.data find-specific-memory-allocation.py
```

Example output:

```output
syscalls__sys_exit_mmap     2 5435674.974718300  1347550 dotnet               
Parameters: addr=0, len=209920, prot=PROT_NONE|PROT_READ, flags=MAP_FILE|MAP_SHARED, fd=56, off=0
ret=78e8c18e7000
	[78e8c7184a27] __mmap
	[78e8c6ea01dd] MapViewOfFileEx
	[78e8c6b430fb] CLRMapViewOfFile
	[78e8c6e7bbd7] FlatImageLayout::FlatImageLayout
	[78e8c6e7aa42] PEImageLayout::LoadFlat
	[78e8c6afa279] PEImage::GetOrCreateLayoutInternal
	[78e8c6af927c] PEImage::GetOrCreateLayout
	[78e8c6b92482] BinderAcquireImport
	[78e8c6db1c43] BINDER_SPACE::AssemblyName::Init
	[78e8c6dac373] BINDER_SPACE::Assembly::Init
	[78e8c6dad838] BINDER_SPACE::AssemblyBinderCommon::GetAssembly
	[78e8c6daf1c8] BINDER_SPACE::AssemblyBinderCommon::BindByTpaList
	[78e8c6dadeac] BINDER_SPACE::AssemblyBinderCommon::BindLocked
	[78e8c6dacb51] BINDER_SPACE::AssemblyBinderCommon::BindByName
	[78e8c6dac7d4] BINDER_SPACE::AssemblyBinderCommon::BindAssembly
	[78e8c6db4a9f] DefaultAssemblyBinder::BindUsingAssemblyName
	[78e8c6a4f67d] AssemblyBinder::BindAssemblyByName
	[78e8c6b91e53] AssemblySpec::Bind
	[78e8c6a4138e] AppDomain::BindAssemblySpec
	[78e8c6af781d] PEAssembly::LoadAssembly
	[78e8c6a5dc04] Module::LoadAssemblyImpl
	[78e8c6a4b6eb] Assembly::FindModuleByTypeRef
	[78e8c6a70072] ClassLoader::LoadTypeDefOrRefThrowing
	[78e8c6b1472d] SigPointer::GetTypeHandleThrowing
	[78e8c6bed9c3] MethodTableBuilder::InitializeFieldDescs
	[78e8c6be7da2] MethodTableBuilder::BuildMethodTableThrowing
	[78e8c6bf99b0] ClassLoader::CreateTypeHandleForTypeDefThrowing
	[78e8c6a713ec] ClassLoader::CreateTypeHandleForTypeKey
	[78e8c6a70e88] ClassLoader::DoIncrementalLoad
	[78e8c6a71d48] ClassLoader::LoadTypeHandleForTypeKey_Body
	[78e8c6a6e2fe] ClassLoader::LoadTypeHandleForTypeKey
	[78e8c6a6f2e4] ClassLoader::LoadTypeDefThrowing
	[78e8c6a6c1f4] ClassLoader::LoadTypeHandleThrowing
	[78e8c6a70293] ClassLoader::LoadTypeDefOrRefThrowing
	[78e8c6b1472d] SigPointer::GetTypeHandleThrowing
	[78e8c6b1558f] SigPointer::GetTypeHandleThrowing
	[78e8c6ac7ed7] CEEInfo::getArgClass
	[78e8c537502d] Compiler::compCompileHelper
	[78e8c5378c9f] jitNativeCode
[.. truncated..]
```

## 4. Best Practices to get good output

### Debug Symbols of common libraries

Ensure that the debug repositories are added and the correct debug symbols are downloaded.

For ubuntu22.04 this would be possible via:
```zsh
apt-get install -y software-properties-common
echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse | \
deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse" | \
tee -a /etc/apt/sources.list.d/ddebs.list
apt install ubuntu-dbgsym-keyring
apt-get update -y
apt-get install -y --no-install-recommends libc6-dbg libssl3-dbgsym openssl-dbgsym libicu70-dbgsym libstdc++6-dbgsym
```

### Debug Symbols of frameworks

Ensure that the debug native symbols for the programming language you are using are also available.

For dotnet 7.0 this would be possible via:
```zsh
dotnet tool install dotnet-symbol --tool-path /opt/tools
/opt/tools/dotnet-symbol /usr/share/dotnet/shared/Microsoft.NETCore.App/7.0.??/*
```

### Dynamic mappings for JIT-compiled code

Languages that support Just-in-Time compilation are able to provide mapping files, that can be automatically used by tools like perf. The files are have to be located under /tmp/ and have the naming schema `perf-<pid>.map`, `perfinfo-<pid>.map` and `jit-<pid>.dump`. It is important to copy those files from the machine that executed the program under trace to the machine that is used to analyze it.

For dotnet 7.0 the generation of JIT-compiled mappings can be activated via:
```zsh
export DOTNET_PerfMapEnabled=1
```

## 5. Additional Resources

* [Brendan Gregg's Blog](https://brendangregg.com/perf.html)
* Manpages [perf(1)](https://www.man7.org/linux/man-pages/man1/perf.1.html), [perf-record(1)](https://www.man7.org/linux/man-pages/man1/perf-record.1.html), [perf-script(1)](https://www.man7.org/linux/man-pages/man1/perf-script.1.html)
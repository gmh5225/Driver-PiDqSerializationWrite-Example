# PiDqSerializationWrite-Example
Thank you busybox10 for sharing knowledge of util functions and "PiDqSerializationWrite" functions with the community.

- https://www.unknowncheats.me/forum/3742948-post13.html

## CPP
```cpp
struct memcpy_structure
{
	void* destination;
	unsigned int    max_size;
	unsigned int    offset;
	unsigned char  pad[0xF];
	unsigned char   error_flag;
};

typedef NTSTATUS(* safe_memcpy)(memcpy_structure* dst, void* src, unsigned __int32 size);
safe_memcpy PiDqSerializationWrite;
```
```cpp
#define NT_HEADER(ModBase) (PIMAGE_NT_HEADERS)((ULONG64)(ModBase) + ((PIMAGE_DOS_HEADER)(ModBase))->e_lfanew)
#define IMAGE_FIRST_SECTION(NtHeader) (PIMAGE_SECTION_HEADER)(NtHeader + 1)
#define SizeAlign(Size) ((Size + 0xFFF) & 0xFFFFFFFFFFFFF000)
#define InRange(x, a, b) (x >= a && x <= b) 
#define GetBits(x) (InRange(x, '0', '9') ? (x - '0') : ((x - 'A') + 0xA))
#define GetByte(x) ((UCHAR)(GetBits(x[0]) << 4 | GetBits(x[1])))

PVOID FindSection(PVOID ModBase, const char* Name, PULONG SectSize)
{
	PIMAGE_NT_HEADERS NT_Header = NT_HEADER(ModBase);
	PIMAGE_SECTION_HEADER Sect = IMAGE_FIRST_SECTION(NT_Header);
	for (PIMAGE_SECTION_HEADER pSect = Sect; pSect < Sect + NT_Header->FileHeader.NumberOfSections; pSect++)
	{
		char SectName[9]; SectName[8] = 0;
		*(ULONG64*)&SectName[0] = *(ULONG64*)&pSect->Name[0];

		if (StrICmp(Name, SectName, true))
		{
			if (SectSize)
			{
				ULONG SSize = SizeAlign(max(pSect->Misc.VirtualSize, pSect->SizeOfRawData));
				*SectSize = SSize;
			}

			return (PVOID)((ULONG64)ModBase + pSect->VirtualAddress);
		}
	}

	return nullptr;
}

PUCHAR FindPatternSect(PVOID ModBase, const char* SectName, const char* Pattern)
{
	ULONG SectSize;
	PUCHAR ModuleStart = (PUCHAR)FindSection(ModBase, SectName, &SectSize);
	PUCHAR ModuleEnd = ModuleStart + SectSize;

	PUCHAR FirstMatch = nullptr;
	const char* CurPatt = Pattern;
	for (; ModuleStart < ModuleEnd; ++ModuleStart)
	{
		bool SkipByte = (*CurPatt == '\?');
		if (SkipByte || *ModuleStart == GetByte(CurPatt))
		{
			if (!FirstMatch) FirstMatch = ModuleStart;
			SkipByte ? CurPatt += 2 : CurPatt += 3;
			if (CurPatt[-1] == 0) return FirstMatch;
		}

		else if (FirstMatch)
		{
			ModuleStart = FirstMatch;
			FirstMatch = nullptr;
			CurPatt = Pattern;
		}
	}

	return nullptr;
}
```
```cpp
NTSTATUS read_memory(PEPROCESS target_process, void* source, void* target, size_t size)
{
	KAPC_STATE ApcState;
	KeStackAttachProcess(target_process, &ApcState);

	memcpy_structure _;
	_.destination = target;
	_.max_size = 0xFFFFFFFF;
	_.offset = 0;
	memset(_.pad, 0, sizeof(_.pad));
	_.error_flag = 0;

	PiDqSerializationWrite(&_, source, size);

	if (_.error_flag)
	{
		KeUnstackDetachProcess(&ApcState);
		return STATUS_UNSUCCESSFUL;
	}

	KeUnstackDetachProcess(&ApcState);
	return STATUS_SUCCESS;
}
```
```cpp
EXTERN_C
NTSTATUS DriverEntry(
	IN PDRIVER_OBJECT DriverObject,
	IN PUNICODE_STRING RegistryPath
) {
	PiDqSerializationWrite = (safe_memcpy)FindPatternSect(GetKernelBase(), "PAGE", "48 89 5C 24 ? 48 89 4C 24 ? 57 48 83 EC 20 41");

	PEPROCESS target_process = NULL;
	if (NT_SUCCESS(PsLookupProcessByProcessId((HANDLE)2948, &target_process)))
	{
		unsigned int buf = 0;
		read_memory(target_process, (void*)0, (void*)&buf, 4);
	}

	return STATUS_UNSUCCESSFUL;
}
```


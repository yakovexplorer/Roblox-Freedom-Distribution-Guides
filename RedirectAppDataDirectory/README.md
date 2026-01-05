By default, Rōblox stores settings in `%LocalAppData%\Roblox`. If you have both modern Rōblox and RFD installed, we'll find file conflicts.

For example, if you use RFD then re-launch modern Rōblox, some settings will be lost (such as automatic translation being re-enabled after you manually disabled it in modern Rōblox).

Let's fix it.

---

## Objective

Look at this method [in the 2016 source](https://github.com/Jxys3rrV/roblox-2016-source-code/blob/4de2dc3a380e1babe4343c49a4341ceac749eddb/App/util/Win/FileSystem.cpp#L27C1-L27C99):

```cpp
boost::filesystem::path getUserDirectory(bool create, FileSystemDir dir, const char *subDirectory)
```

The `dir` argument is interesting to me. Depending on its value, the result calculates the file-save path for images, videos, and even the EXE's directory itself.

**Our goal is to alter execution flow for when the `dir` argument is `DirAppData` whilst keeping all other flows intact.**

```cpp
enum FileSystemDir
{
    DirAppData = 0,
    DirPicture,
    DirVideo,
    DirExe
};
```

By default, if you are running a Rōblox executable at `C:\Users\USER\Projects\FilteringDisabled\Roblox\v348\Server\*.exe`, the expected result will be a path at `C:\Users\USER\AppData\Local\Roblox\`.

We want these files to be saved at a relative path of the user's choosing. It can be controlled by a fast variable named `FStringUS20608`.

Looking through other places in the codebase which call the method named `getUserDirectory`, I found [this](https://github.com/Jxys3rrV/roblox-2016-source-code/blob/4de2dc3a380e1babe4343c49a4341ceac749eddb/CSG/CSGKernel.cpp#L190):

```cpp
boost::filesystem::path path = RBX::FileSystem::getUserDirectory(true, RBX::DirAppData, "logs");
```

Notice here the interesting string `"logs"`.

## Using x32dbg

Let's open up x32dbg using the 2018E (v348) `RCCService.exe`.

Then search for user strings.

![alt text](image.png)

If you get multiple results, clicking on just one will suffice.

```
004973D5 | 68 64E50B01              | push rccservice.10BE564                                        | 10BE564:"logs"
004973DA | 6A 00                    | push 0                                                         |
004973DC | 8D45 D8                  | lea eax,dword ptr ss:[ebp-28]                                  |
004973DF | C745 FC 00000000         | mov dword ptr ss:[ebp-4],0                                     |
004973E6 | 6A 01                    | push 1                                                         |
004973E8 | 50                       | push eax                                                       |
004973E9 | E8 C2040E00              | call rccservice.5778B0                                         |
```

The very next function call (in this case) is to `robloxplayerbeta.5778B0`. Let's look inside!

### Finding Available Code Regions

The `DirAppData` enum has _exclusive_ code coverage in **two** areas, according to the 2016 source-code archive.

#### (C1)

```cpp
switch (dir)
{
	case DirAppData:
		hr = SHGetFolderPathAndSubDirW(NULL, CSIDL_LOCAL_APPDATA | flags, NULL, SHGFP_TYPE_CURRENT, robloxDir.native().c_str(), pathBuffer);
		break;
	case DirPicture:
		hr = SHGetFolderPathAndSubDirW(NULL, CSIDL_MYPICTURES | flags, NULL, SHGFP_TYPE_CURRENT, robloxDir.native().c_str(), pathBuffer);
		break;
	case DirVideo:
		hr = SHGetFolderPathAndSubDirW(NULL, CSIDL_MYVIDEO | flags, NULL, SHGFP_TYPE_CURRENT, robloxDir.native().c_str(), pathBuffer);
		break;
	default:
		RBXASSERT(false);
		hr = S_FALSE;
		break;
}
```

This region has some of the same call to `SHGetFolderPathAndSubDirW`, which x32dbg conveniently denotes for us below:

![alt text](image-1.png)

Notice that the address pointing to `SHGetFolderPathAndSubDirW` is stored in register `ebx` at instruction `00577A5C`.

Why? The C++ code shows `SHGetFolderPathAndSubDirW` being referenced _thrice_, since there are three switch-cases which use it. This is definitely a special compiler optimisation from MSVC++.

The subsequent `je` and `sub` statements must correspond to the switch-case. It should be safe to assume that the `eax` register being processed is the enum argument `dir` in C++.

From earlier, we determined that `dir` should be equal to zero. So we want to trace x86 execution for when `eax` is _originally_ equal to `0`. This leads us to the highlighted region: from `00577AAE` to `00577AC3`.

Therefore, no other enum values for `dir` can reach this code segment. It should be safe to `nop` the entire region.

![alt text](image-2.png)

The `jmp` immediately after the highlighted region is a bridge for the _next_ available region of code to discuss. Fortunately, these two code regions are near each other.

#### (C2)

```cpp
if ((hr!=S_OK) && (dir == DirAppData))
{
    // Try this one:
    hr = SHGetFolderPathAndSubDirW(NULL, CSIDL_COMMON_APPDATA | flags, NULL, SHGFP_TYPE_CURRENT, robloxDir.native().c_str(), pathBuffer);
}
```

This code's execution flow depends on if the previous region yields failure via a C++ variable `hr`. Since `SHGetFolderPathAndSubDirW` is being used again, it is reasonable to assume that `call ebx`.

![alt text](image-3.png)

Combining the two segments together, we can trace this execution graph:

```mermaid
---
config:
  themeVariables:
    fontFamily: Monospace
  flowchart:
    markdownAutoWrap: false
    wrappingWidth: 6767
---
flowchart TB
 subgraph subGraph0["Switch-case statement per (C1)"]
        3.5["ASSERT FALSE"]
        3["mov eax,1<br>jmp rccservice.577AD3"]
        2["sub eax,1<br>je rccservice.577A94"]
        1["sub eax,1<br>je rccservice.577A94"]
        0["mov ebx,dword ptr ds:[]<br>mov eax,edi<br>sub eax,0<br>je rccservice.577AAE"]
        0.5["cmp dword ptr ss:[ebp-10],8<br>lea eax,dword ptr ss:[ebp-24]<br>lea ecx,dword ptr ss:[ebp-250]<br>cmovae eax,dword ptr ss:[ebp-24]<br>push ecx<br>push eax<br>mov eax,esi<br>or eax,1C"]
        1.5["cmp dword ptr ss:[ebp-10],8<br>lea eax,dword ptr ss:[ebp-24]<br>lea ecx,dword ptr ss:[ebp-250]<br>cmovae eax,dword ptr ss:[ebp-24]<br>push ecx<br>push eax<br>mov eax,esi<br>or eax,27<br>jmp rccservice.577AC6"]
        2.5["cmp dword ptr ss:[ebp-10],8<br>lea eax,dword ptr ss:[ebp-24]<br>lea ecx,dword ptr ss:[ebp-250]<br>cmovae eax,dword ptr ss:[ebp-24]<br>push ecx<br>push eax<br>mov eax,esi<br>or eax,E<br>jmp rccservice.577AC6"]
        C["push 0<br>push 0<br>push eax<br>push 0<br>call ebx"]
  end
 subgraph subGraph1["Backup call to CSIDL_COMMON_APPDATA per (C2)"]
        J["or eax,1C<br>push 0<br>push 0<br>push eax<br>push 0<br>call ebx<br>test eax,eax<br>je rccservice.577B0A<br>test edi,edi<br>jne rccservice.577AF3"]
        C2["mov esi,dword ptr ss:[ebp+8]<br>lea eax,dword ptr ss:[ebp-250]<br>push eax<br>mov ecx,esi<br>call rccservice.48F430"]
        B2["test eax,eax<br>je rccservice.577B0A"]
        C0["cmp dword ptr ss:[ebp-10],8<br>lea ecx,dword ptr ss:[ebp-250]<br>push ecx<br>lea eax,dword ptr ss:[ebp-24]<br>cmovae eax,dword ptr ss:[ebp-24]<br>or esi,23<br>push eax<br>push edi<br>push edi<br>push esi<br>push edi<br>call ebx"]
        C1["mov esi,dword ptr ss:[ebp+8]<br>mov ecx,esi<br>push 0<br>push rccservice.10BB8CE<br>call rccservice.48CC90<br>jmp rccservice.577B1B"]
  end
    0 -- jump not taken --> 1
    1 -- jump not taken --> 2
    2 -- jump not taken --> 3
    3 --> 3.5
    BEGIN["BEGIN"] --> 0
    0 -- jump taken --> 0.5
    1 -- jump taken --> 1.5
    2 -- jump taken --> 2.5
    0.5 --> C
    1.5 --> C
    2.5 --> C
    C --> J
    J -- "eax == 0 (success)" --> C2
    J -- "eax != 0 &amp;&amp; edi == 0<br>(fail, non-AppData dir)" --> B2
    J -- "eax != 0 &amp;&amp; edi == 0<br>(fail, AppData dir)" --> C0
    C0 --> B2
    B2 -- jump not taken (fail) --> C2
    B2 -- jump taken (success) --> END["END"]
    C2 --> END
    C1 --> END
```

There are two registers being checked in x86: `eax` and `edi`. We know that _one_ of them corresponds with the C++ variable `dir`. It is safe to eliminate `eax` since both comparisons are made _immediately_ after a function call at `00577ACD`.

It is therefore safe to completely skip the _second_ condition and make that jump unconditional. This leave us with plenty more bits to `nop`.

```mermaid
---
config:
  themeVariables:
    fontFamily: Monospace
  flowchart:
    markdownAutoWrap: false
    wrappingWidth: 6767
---
flowchart TB
    BEGIN["BEGIN"] --> 0["mov ebx,dword ptr ds:[]<br>mov eax,edi<br>sub eax,0<br>je rccservice.577AAE"]
    0 -- jump is taken --> 1["nop (x22)<br>...<br>jmp rccservice.577AD5"]
    1 --> 2["nop (x32)<br>...<br>jmp rccservice.577B0A"]
    2 --> 3["mov esi,dword ptr ss:[ebp+8]<br>lea eax,dword ptr ss:[ebp-250]<br>push eax<br>mov ecx,esi<br>call rccservice.48F430"]
    3 --> END["END"]
```

![alt text](image-4.png)

To allow more space for our own own instructions, I've had to move some of the jumps in addition to making them unconditional. For example,

```diff
-00577AC0 | 50                       | push eax
-00577AC1 | 8BC6                     | mov eax,esi
-00577AC3 | 83C8 1C                  | or eax,1C
+00577AC0 | 90                       | nop
+00577AC1 | 90                       | nop
+00577AC2 | 90                       | nop
+00577AC3 | 90                       | nop
+00577AC4 | EB 0F                    | jmp rccservice.577AD5
00577AC6 | 6A 00                    | push 0
```

```diff
00577AD1 | 74 37                    | je rccservice.577B0A
-00577AD3 | 85FF                     | test edi,edi
-00577AD5 | 75 1C                    | jne rccservice.577AF3
+00577AD3 | EB 22                    | jmp rccservice.577AF7
+00577AD5 | 90                       | nop
+00577AD6 | 90                       | nop
...
-00577AF3 | 85C0                     | test eax,eax
-00577AF5 | 74 13                    | je rccservice.577B0A
+00577AF3 | 90                       | nop
+00577AF4 | 90                       | nop
+00577AF5 | EB 13                    | jmp rccservice.577B0A
00577AF7 | 8B75 08                  | mov esi,dword ptr ss:[ebp+8]
```

### Deciding What Code to Assemble

By looking at the assembly, we find that:

- **`ebp - 0x250`** is **the first character** of a `wchar_t[]` buffer used in SHGetFolderPathAndSubDirW branches; we redirect "subDirectory" to this address in our patches for AppData.
- **`ebp + 0x14`** is **a pointer** to the `const char* subDirectory` function argument.

Therefore, I want the buffer at `ebp - 0x250` to be filled like the following:

- a `wchar*` string `L"../../logs"` for when `subDirectory` is equal to narrow-character string `"logs"`.

- a `wchar*` string `L"../../"` for when `subDirectory` an empty string or a null pointer.

After much trial and error, I devised this clever 40-byte-long routine which does exactly that:

```x86
lea edi,dword ptr ss:[ebp-250]
mov eax,2F002E
mov dword ptr ds:[edi+8],eax
ror eax,10
mov dword ptr ds:[edi+4],eax
dec eax
mov dword ptr ds:[edi],eax

lea eax,dword ptr ds:[edi+C]
push es
push dword ptr ss:[ebp+14]
push eax
call dword ptr ds:[<mbstowcs>]
add esp,C
```

All I need to do is fill the nops in with code assembled and sequenced from the instructions above.

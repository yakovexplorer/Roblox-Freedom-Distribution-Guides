# Force 2017- RCCServices to fetch FFlags

For some reason, Roblox decided that the "RCCService" FFlags should be only fetched when a Settings Key is present in the registry. We should make it fetch FFlags no matter what. This guide is useful for launchers, and maybe some revivals. This guide is useless for 2018+ RCCServices, as they already fetch FFlags no matter what. We need a 2017- RCCService today.

# Steps
## 1. Search for "Read settings key: %s" in string references
Open x32dbg with your RCCService executable. Then click on "Symbols".

![Symbols image](x32dbg_NK5YxAOPM2.png)

Find your executable name and double-click on it. You must have been dropped back to the CPU tab. If not, then manually go to the CPU tab.<br>
Then click the little "Az" icon in the top bar.

![Point to Az](x32dbg_iFaWchMD1a.png)

Then you will be in the "References" tab. Wait until the progress bar at the bottom finishes, and type in the "Search:" field the phrase "Read settings key: %s", like this:

![References](x32dbg_Kor4k5yy0O.png)

Double click on the reference found.

## 2. Assembly
The hardest part. You might see a lot of instructions like these now:

![Assembly](x32dbg_v16iR1YUwO.png)

First things first, find a `js` instruction above these 6:

```
00602550 | 8B45 F0                  | mov eax,dword ptr ss:[ebp-10]           |
00602553 | 8D4D CC                  | lea ecx,dword ptr ss:[ebp-34]           |
00602556 | C68405 C4FEFFFF 00       | mov byte ptr ss:[ebp+eax-13C],0         |
0060255E | 8D85 C4FEFFFF            | lea eax,dword ptr ss:[ebp-13C]          |
00602564 | 50                       | push eax                                |
00602565 | E8 D6FBFBFF              | call rccservice.5C2140                  |
```

Select it, press space, change `js` to `jmp` and press OK. It must look like this now:

![js to jmp](x32dbg_pSA1G49CzI.png)

After you have done that, find this instruction:

![cmp](x32dbg_4ok3OXV8kq.png)

Select it, press space, change `0x00` to `edi` like the image below and press OK.

**WARNING: Make sure you have "Fill with NOP's" enabled, otherwise it will eat the "je" instruction!**

![edit](x32dbg_GQvDhqF59z.png)

You are done now. Press the band aid icon in the topbar,

![band aid](x32dbg_V6EPARt4wV.png)

click "Patch" and save the executable.

# Patch export is avaliable [here](rccservice.1337).

Guide made by SomeoneInTheWorld (@yakovexplorer on Github).
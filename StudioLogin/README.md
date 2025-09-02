## How to find the patch?

## Background

I found a version of Studio from late 2021 ([v493.1.15175](https://archive.org/details/roblox-version-1fca050e38094184)) with a login patch applied in x86. There was a change from `jnz` to `jz` at file address `0x002B951C`. This patch was location near a unique string reference to `"Studio.App.AutoSaveDialog.OpenRobloxFile"`. This fix, however, can't be reproduced in my target versions of v348 and v463.

## Preliminary Findings

When a login routine is successful, you expect some log message to show up (at least conditionally).

Let's try `"[FLog::LoginManager] LoginManager::loginSuccess"`.

Using x64dbg on v463 of Studio, we get one user-module reference at `00000001403AF84B`.

```
00000001403AF82D | CC                       | int3                                                    |
00000001403AF82E | CC                       | int3                                                    |
00000001403AF82F | CC                       | int3                                                    |
00000001403AF830 | 48:83EC 78               | sub     rsp, 0x78                                       |
00000001403AF834 | 48:C74424 20 FEFFFFFF    | mov     qword ptr ss:[rsp + 0x20], 0xFFFFFFFFFFFFFFFE   |
00000001403AF83D | 0FB60D 34E6F002          | movzx   ecx, byte ptr ds:[0x1432BDE78]                  |
00000001403AF844 | 84C9                     | test    cl, cl                                          |
00000001403AF846 | 74 0F                    | je      robloxstudiobeta.1403AF857                      |
00000001403AF848 | 45:33C0                  | xor     r8d, r8d                                        |
00000001403AF84B | 48:8D15 AE560E02         | lea     rdx, qword ptr ds:[0x142494F00]                 | 0000000142494F00:"[FLog::LoginManager] LoginManager::loginSuccess"
00000001403AF852 | E8 891F7E01              | call    robloxstudiobeta.141B917E0                      |
00000001403AF857 | 66:0F6F05 D1C6D301       | movdqa  xmm0, xmmword ptr ds:[0x1420EBF30]              |
```

Go to the top of the function call at `00000001403AF830`. Then search for xrefs. I found one at `00000001403B435E`.

```
00000001403B433E | CC                       | int3
00000001403B433F | CC                       | int3
00000001403B4340 | 4C:8BC2                  | mov     r8, rdx
00000001403B4343 | 85C9                     | test    ecx, ecx
00000001403B4345 | 74 1C                    | je      robloxstudiobeta.1403B4363
00000001403B4347 | 83E9 01                  | sub     ecx, 0x1
00000001403B434A | 74 0E                    | je      robloxstudiobeta.1403B435A
00000001403B434C | 83F9 01                  | cmp     ecx, 0x1
00000001403B434F | 75 24                    | jne     robloxstudiobeta.1403B4375
00000001403B4351 | 48:8B4424 28             | mov     rax, qword ptr ss:[rsp + 0x28]
00000001403B4356 | C600 00                  | mov     byte ptr ds:[rax], 0x0
00000001403B4359 | C3                       | ret
00000001403B435A | 48:8D4A 10               | lea     rcx, qword ptr ds:[rdx + 0x10]
00000001403B435E | E9 CDB4FFFF              | jmp     robloxstudiobeta.1403AF830
00000001403B4363 | 4D:85C0                  | test    r8, r8
00000001403B4366 | 74 0D                    | je      robloxstudiobeta.1403B4375
00000001403B4368 | BA 18000000              | mov     edx, 0x18
00000001403B436D | 49:8BC8                  | mov     rcx, r8
00000001403B4370 | E9 6BA17D01              | jmp     <robloxstudiobeta.rbxDeallocate>
00000001403B4375 | C3                       | ret
00000001403B4376 | CC                       | int3
```

Go to the top of the function call at `00000001403B4340`. Then search for xrefs. I found one at `00000001403B4E06`.

```
00000001403B4DF2 | E8 A9967D01              | call    robloxstudiobeta.141B8E4A0
00000001403B4DF7 | 48:8945 F0               | mov     qword ptr ss:[rbp - 0x10], rax
00000001403B4DFB | 48:85C0                  | test    rax, rax
00000001403B4DFE | 74 13                    | je      robloxstudiobeta.1403B4E13
00000001403B4E00 | C700 01000000            | mov     dword ptr ds:[rax], 0x1
00000001403B4E06 | 48:8D0D 33F5FFFF         | lea     rcx, qword ptr ds:[0x1403B4340]
00000001403B4E0D | 48:8948 08               | mov     qword ptr ds:[rax + 0x8], rcx
00000001403B4E11 | EB 03                    | jmp     robloxstudiobeta.1403B4E16
00000001403B4E13 | 49:8BC7                  | mov     rax, r15
00000001403B4E16 | 48:895C24 40             | mov     qword ptr ss:[rsp + 0x40], rbx
00000001403B4E1B | 4C:897C24 38             | mov     qword ptr ss:[rsp + 0x38], r15                                           [rsp+38]:public: static void __cdecl QMetaObject::activate(class QObject *, int, int, void **)+4C6
00000001403B4E20 | C74424 30 01000000       | mov     dword ptr ss:[rsp + 0x30], 0x1
00000001403B4E28 | 48:894424 28             | mov     qword ptr ss:[rsp + 0x28], rax
00000001403B4E2D | 4C:897C24 20             | mov     qword ptr ss:[rsp + 0x20], r15
00000001403B4E32 | 4C:8BCE                  | mov     r9, rsi
00000001403B4E35 | 4C:8D45 50               | lea     r8, qword ptr ss:[rbp + 0x50]
00000001403B4E39 | 48:8BD6                  | mov     rdx, rsi
00000001403B4E3C | 48:8D4D 58               | lea     rcx, qword ptr ss:[rbp + 0x58]
00000001403B4E40 | FF15 8A19D101            | call    qword ptr ds:[<private: static class QMetaObject::Connection __cdecl QObject::connectImpl(class QObj...
```

Note the call to `QObject::connectImpl` at address `00000001403B4E40`.

## What Else?

Let's check potential log messages. It would make sense that we search for `Login` or the like. Let's search for user references to `FLog::LoginManager`:

```
00000001403AD96A lea rdx,qword ptr ds:[142495230] 0000000142495230 "[FLog::LoginManager] LoginManager::logOut networkReply was NULL."
00000001403ADC03 lea rdx,qword ptr ds:[142495480] 0000000142495480 "[FLog::LoginManager] Logged in user with hash '%s' with id with hash '%s'"
00000001403ADD0F lea rdx,qword ptr ds:[1424954D0] 00000001424954D0 "[FLog::LoginManager] Logged in user '%s' with id '%s'"
00000001403ADE4C lea rdx,qword ptr ds:[142494E18] 0000000142494E18 "[FLog::LoginManager] LoginManager::captchaNeeded"
00000001403AECBB lea rdx,qword ptr ds:[142494E90] 0000000142494E90 "[FLog::LoginManager] LoginManager::onTwoStepVerificationWidgetShown"
00000001403AF84B lea rdx,qword ptr ds:[142494F00] 0000000142494F00 "[FLog::LoginManager] LoginManager::loginSuccess"
00000001403AFAF8 lea rdx,qword ptr ds:[142495210] 0000000142495210 "[FLog::LoginManager] %s"
00000001403AFC86 lea rdx,qword ptr ds:[142495210] 0000000142495210 "[FLog::LoginManager] %s"
00000001403AFCC8 lea rdx,qword ptr ds:[142495790] 0000000142495790 "[FLog::LoginManager] Verified two step verification code."
00000001403B0194 lea rdx,qword ptr ds:[1424950F8] 00000001424950F8 "[FLog::LoginManager] LoginManager::fetchUserId"
00000001403B0767 lea rdx,qword ptr ds:[1424951B8] 00000001424951B8 "[FLog::LoginManager] LoginManager::fetchUsername"
00000001403B0DFC lea rdx,qword ptr ds:[142495210] 0000000142495210 "[FLog::LoginManager] %s"
00000001403B0EA3 lea rdx,qword ptr ds:[142495338] 0000000142495338 "[FLog::LoginManager] LoginManager successfully logged out."
00000001403B0EE9 lea rdx,qword ptr ds:[142495338] 0000000142495338 "[FLog::LoginManager] LoginManager successfully logged out."
00000001403B0F46 lea rdx,qword ptr ds:[142495230] 0000000142495230 "[FLog::LoginManager] LoginManager::logOut networkReply was NULL."
00000001403B3BBE lea rdx,qword ptr ds:[142495550] 0000000142495550 "[FLog::LoginManager] Two step verification required."
00000001403B4112 lea rdx,qword ptr ds:[142495518] 0000000142495518 "[FLog::LoginManager] Incorrect username or password."
00000001403B546F lea rdx,qword ptr ds:[1424952D8] 00000001424952D8 "[FLog::LoginManager] LoginManager::logOut()"
00000001403B5D3E lea rdx,qword ptr ds:[142495378] 0000000142495378 "[FLog::LoginManager] LoginManager::onAuthenticationChanged(%d)"
00000001403B5FAB lea rdx,qword ptr ds:[142495850] 0000000142495850 "[FLog::LoginManager] LoginManager::onExternalCaptchaLinkClicked()"
00000001403B623B lea rdx,qword ptr ds:[1424958C0] 00000001424958C0 "[FLog::LoginManager] LoginManager::onForgotPasswordClicked()"
00000001403B636F lea rdx,qword ptr ds:[142495670] 0000000142495670 "[FLog::LoginManager] LoginManager::onInitialAuthenticationDone(%d)"
00000001403B695A lea rdx,qword ptr ds:[142495D70] 0000000142495D70 "[FLog::LoginManager] Could not log in. Got message '%s'"
00000001403B6A2B lea rdx,qword ptr ds:[142495920] 0000000142495920 "[FLog::LoginManager] LoginManager::onSignUpClicked()"
00000001403B6E40 lea rdx,qword ptr ds:[142495210] 0000000142495210 "[FLog::LoginManager] %s"
00000001403B783A lea rdx,qword ptr ds:[142495210] 0000000142495210 "[FLog::LoginManager] %s"
00000001403B81E7 lea rdx,qword ptr ds:[142495800] 0000000142495800 "[FLog::LoginManager] LoginManager::twoStepVerificationStartOverClicked()"
00000001405F2EBE lea rdx,qword ptr ds:[1424F74A8] 00000001424F74A8 "[FLog::LoginManager] Login failure reply data %s"
00000001405F3283 lea rdx,qword ptr ds:[142495210] 0000000142495210 "[FLog::LoginManager] %s"
00000001405FD835 lea rdx,qword ptr ds:[1424F9FA0] 00000001424F9FA0 "[FLog::LoginManager] LoginRequestAuth::DEPRECATED_handleError(%p)"
00000001405FD984 lea rdx,qword ptr ds:[1424F9FF0] 00000001424F9FF0 "[FLog::LoginManager] LoginRequestAuth::DEPRECATED_handleError() - got error from server [status code = %d] [body = %s]"
00000001405FDBFF lea rdx,qword ptr ds:[1424F9D18] 00000001424F9D18 "[FLog::LoginManager] LoginRequestAuth::onReplyFinished(%p)"
00000001405FE59D lea rdx,qword ptr ds:[1424F9E68] 00000001424F9E68 "[FLog::LoginManager] LoginRequestAuth::handleError(%p)"
00000001405FE6C9 lea rdx,qword ptr ds:[1424F9EF0] 00000001424F9EF0 "[FLog::LoginManager] LoginRequestAuth::handleError() got error from server [status code = %d] [body = %s]"
00000001405FE9DA lea rdx,qword ptr ds:[1424FA070] 00000001424FA070 "[FLog::LoginManager] LoginRequestAuth::handleTwoStepVerification"
00000001405FEDB6 lea rdx,qword ptr ds:[1424F9D18] 00000001424F9D18 "[FLog::LoginManager] LoginRequestAuth::onReplyFinished(%p)"
00000001405FFF3B lea rdx,qword ptr ds:[142495210] 0000000142495210 "[FLog::LoginManager] %s"
0000000140600079 lea rdx,qword ptr ds:[142495210] 0000000142495210 "[FLog::LoginManager] %s"
00000001406006DA lea rdx,qword ptr ds:[142495210] 0000000142495210 "[FLog::LoginManager] %s"
0000000140600818 lea rdx,qword ptr ds:[142495210] 0000000142495210 "[FLog::LoginManager] %s"
00000001407C63AF lea rdx,qword ptr ds:[142569200] 0000000142569200 "[FLog::LoginManager] Could not parse login error response. The errors field is not an array."
00000001407C6431 lea rdx,qword ptr ds:[142569260] 0000000142569260 "[FLog::LoginManager] Could not parse login error response. Could not read error code."
00000001407C649C lea rdx,qword ptr ds:[1425692C0] 00000001425692C0 "[FLog::LoginManager] Could not parse login error response. Could not read error message."
```

I added a breakpoint to each result.

These strings are constructed in a way that they print to a log file. And we know that `FLog` messages conditionally print pursuant to an FFlag. So I followed one of the results and looked for any jump-type statements prior:

```
00000001403AFCBA | 0FB60D B7E1F002          | movzx   ecx, byte ptr ds:[0x1432BDE78]                                                                       |
00000001403AFCC1 | 84C9                     | test    cl, cl                                                                                               |
00000001403AFCC3 | 74 0F                    | je      robloxstudiobeta.1403AFCD4                                                                           |
00000001403AFCC5 | 45:33C0                  | xor     r8d, r8d                                                                                             |
00000001403AFCC8 | 48:8D15 C15A0E02         | lea     rdx, qword ptr ds:[0x142495790]                                                                      | 0000000142495790:"[FLog::LoginManager] Verified two step verification code."
00000001403AFCCF | E8 0C1B7E01              | call    robloxstudiobeta.141B917E0                                                                           |
00000001403AFCD4 | 48:C745 CF 00000000      | mov     qword ptr ss:[rbp - 0x31], 0x0                                                                       | [rbp-31]:__RTDynamicCast+61
00000001403AFCDC | 48:C745 D7 0F000000      | mov     qword ptr ss:[rbp - 0x29], 0xF                                                                       |
```

In x64dbg, we follow the address `0x1432BDE78` in the memory dump. I set the value of the byte from `00` to `01`. An _alternative_ would be to do something like setting `FLogLoginManager` to `true` in `ClientSettings.json`.

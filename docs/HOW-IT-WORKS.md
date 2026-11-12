# How it works: technical deep dive

This is the long version of the [README](../README.md#-how-it-was-done). It covers the game's network protocol, each patch in detail, and how the APK is put together.

- [What's inside the APK](#whats-inside-the-apk)
- [The protocol](#the-protocol)
- [The private server](#the-private-server)
- [Patch 1: the popups](#patch-1-the-popups)
- [Patch 2: the START crash](#patch-2-the-start-crash)
- [Rebuilding without breaking anything](#rebuilding-without-breaking-anything)

---

## What's inside the APK

```
ArelWars2-64bit-server-3.apk          package com.zettabridge.launcher, arm64-v8a
├── lib/arm64-v8a/libzbridge.so       the translation layer (64-bit)
├── classes.dex                       ZettaBridge launcher (Java)
├── classes2.dex                      private server + Application hook
├── assets/zb/sysroot/system/…        32-bit linker, libc, libm, libc++, libz, liblog, libdl
├── assets/zb/guest/lib/…             32-bit stand-ins: EGL, GLESv1_CM, GLESv2, android, jnigraphics
├── assets/zb/host/libzbproxy.so      64-bit side of those stand-ins
└── assets/bundled/game.apk           Arel Wars 2 1.0.7 (Gamevil), patched, stored uncompressed
```

The game APK keeps its own package name inside (`com.gamevil.ArelWars2.global`, target SDK 14). The launcher imports it on first run and again whenever the launcher's version code or install time changes, so an update always brings the new game files with it.

## The protocol

All of this was read from `libgameDSO.so`, which still carries its C++ symbols (`CPdNet`, `CGsNetCore`). The game opens **one TCP connection per action**.

```
packet   = u16le size (whole packet, header included) | u16le command | body
answer   = command + 1, body[0] = signed result (>= 0 ok, < 0 error)
```

There's no encryption or base64, because `CPdNet` was built with both turned off. The receive buffer is `0x2800` bytes.

Most requests start with a "phone block" (`phone[64] androidId[64] deviceId[64]`). After connecting, the game normally sends `CONNECT_V2` first and then the command for the action. A few link types skip `CONNECT_V2` and send their command directly. While a connection stays open, the game pings with `0x7000` every 25 seconds.

**Error codes the game reacts to:**
- `-14` shows "no registered user".
- `-6` and `-7` show the "hacking user" popup.
- Anything below `-49` makes the game send `GET_EXCEPTION_MESSAGE` (`0x104`) and show the text the server sends back. The private server uses this (`-100`) to say politely that a feature isn't available.

| Command | Name | Answer the game expects |
|---|---|---|
| `0x314` | CONNECT_V2 | Result, status flags, backup size, user id, mail counter, PvP values, cash, name[48] |
| `0x302` | REGISTER_USER_ID | Result (request carries name[48]) |
| `0x308` | CASH_INCREASE_DECREASE | Result, u32 cash |
| `0x30e` | GET_BALANCE | Result, reward / HP / wave-timer percentages, three empty lists |
| `0x304` | GET_MESSAGE | Result, text[800] |
| `0x30a` | GET_INFORM | Result, count, entries of 58 bytes |
| `0x316` | DAY_REWARD_V3 | Result, popup flag, day, reward list, u32 cash, u32 seconds |
| `0x30c` / `0x31a` | STATISTICS_DATA (V2) | Result, hours until next upload (0 = never) |
| `0x204` | GET_BUY_ITEM_LIST | Result, up to 6 gold packs, up to 6 cash packs |
| `0x200`, `0x202`, `0x206`, `0x106` | Purchases | Result, u32 cash |
| `0x500` | GET_MAIL_LIST | Result, page flag, count, entries |
| `0x502` | OPEN_MAIL | 446 bytes, cash at offset 1 |
| `0x900` / `0x902` | UPLOAD_BACKUP (+ confirm) | Result per 5000-byte chunk |
| `0xa00` / `0xa02` | DOWNLOAD_BACKUP (+ confirm) | Result, more flag, length, data |
| `0x310` | GET_FR_ACCEPT | Result, flag, count |

PvP (`0x400`, `0x700`–`0x712`, `0xb00`–`0xb04`), friends (`0x800`–`0x80e`) and sending mail (`0x600`/`0x602`) aren't decoded yet.

**Cash** is the u32 at `SysMenu+0x25c`. It always comes from the server, and the game caps it at 99,999,999. **Gold** is local (`CInventory+8`, max 9,999,999).

## The private server

- It's plain Java (Java 8 level, no libraries), compiled with `d8 --min-api 21` into `classes2.dex`.
- The manifest's application class `.ZbApplication` was renamed in place to `.AwApplication`. The new name has the same length, so the binary manifest's string pool didn't need rebuilding. `AwApplication` extends `ZbApplication` and only adds one call that starts the server thread.
- The app runs in more than one process (the launcher and `:guest`). Whichever process binds `127.0.0.1:17001` first serves, and the others retry every 3 seconds. That way the server keeps running even if one process is restarted.
- Accounts are keyed by device. The first one gets the GM name, and later ones are named `Player<uid>`.
- The server was developed against a small Python client that plays the game's side of every message.

## Patch 1: the popups

Both popups were found by decompiling the game's `classes.dex` with apktool and following the dialog resource IDs back to their callers.

**Network Error / Network Not Available.** `com.gamevil.lib.GvActivity` calls `doJellyBeen()` and `doTorchwood()` from `onResume()`. Each starts an `AsyncTask` from `com.gamevil.lib.profile.GvTorchwood`, which calls Gamevil and shows the dialog when that fails. Both methods are now empty:

```smali
.method public doJellyBeen()V
    .locals 0
    return-void
.end method
```

`doTorchwood()` got the same treatment.

**"Can't connect to Market… out of date" / "billing not supported".** These are dialogs 1 and 2 of `ArelWars2Launcher`, raised by the old Android Market billing v1 service check, in `ArelWars2Launcher` and in `ArelWars2Launcher$Nexus2PurchaseObserver`. The three `showDialog()` calls for them were removed. Dialog 4 ("purchase complete") is untouched.

## Patch 2: the START crash

The launcher records a run report when the game crashes. The report for this crash showed:

```
SIGSEGV in guest code, libgameDSO.so + 0xb1ae6
→ CPdMap::SetupScrollData   (called from CPdMap::Load)
```

`SetupScrollData` was reading map-layer data that had never been loaded. The decision to skip loading is made in `CMapLayerData::PreLoad`:

```asm
0xa9a44   00 2b    cmp  r3, #0
0xa9a46   1d dd    ble  0xa9a84      ; skip the layer if pointer <= 0
```

`ble` treats the pointer as a **signed** number. On the phones the game was written for, heap pointers were low addresses and always positive, so only NULL took the branch. Under the translation layer the 32-bit guest's memory is mapped above `0x80000000`. Every valid pointer then reads as negative, so every layer was skipped.

Changing the condition from `le` (`0xD`) to `eq` (`0x0`) turns it into a plain null check:

```asm
0xa9a46   1d d0    beq  0xa9a84      ; skip the layer only if pointer == NULL
```

That's one byte, at file offset `0xa9a47`, from `0xDD` to `0xD0`. It was applied to `lib/armeabi/libgameDSO.so` and `lib/armeabi-v7a/libgameDSO.so`, which have the same bytes at that offset. A byte-by-byte comparison against the original libraries shows no other difference.

> If other places in the game make the same signed-pointer assumption, they'll show up the same way: a crash report pointing into `libgameDSO.so`. Each one is a similar one-byte fix.

## Rebuilding without breaking anything

Old Android games are fussy about their APK layout. Native libraries and some assets have to stay **uncompressed** to be loaded straight from the APK. Re-packing everything with a tool would quietly change that. So the rebuild was done surgically:

1. Decode the game with apktool, patch the smali, and rebuild. Only the resulting `classes.dex` is used.
2. Rebuild the game APK in Python with `zipfile`, copying every entry from the **original** APK with its original compression type. Only `classes.dex` and the two `libgameDSO.so` files are swapped in.
3. Put that game APK into the launcher as `assets/bundled/game.apk`, **stored** (not deflated), and remove the old signature files from `META-INF`.
4. Zipalign, then sign with the project key (APK signature schemes v2 and v3). It's the same key as every earlier build, so the update installs in place.

The key and its password are kept off GitHub on purpose. Anyone with them could publish an "update" that installs over your copy.

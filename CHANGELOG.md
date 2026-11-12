# Changelog

All builds are signed with the same key and install over each other, keeping your save.

## Build 3 · `ArelWars2-64bit-server-3.apk` (latest)

- **Fixed:** the game crashed after pressing **START**. A signed pointer check in the map loader (`CMapLayerData::PreLoad`) treated the high memory addresses used under the translation layer as "no data". The check is now a plain null test, which is one byte changed in `libgameDSO.so` (`ble` → `beq` at `0xa9a47`, in both `armeabi` and `armeabi-v7a`).

## Build 2 · `ArelWars2-64bit-server-2.apk`

- **Removed:** the "Network Error" and "Network Not Available" popups. Gamevil's `GvTorchwood` analytics no longer runs on resume.
- **Removed:** the "Can't connect to Market… out of date" and "billing not supported" popups from the old Android Market billing check.
- Only `classes.dex` of the game changed, and every other file in the game APK is byte-for-byte the original.

## Build 1 · `ArelWars2-64bit-server.apk`

- **Added:** a private server inside the app on `127.0.0.1:17001`. It handles accounts, cash, the shop list, daily reward, messages, mail and save backup.
- **Added:** a GM account. The first account on the phone gets a fixed 9,999,999 cash.

## 64-bit port · `ArelWars2-64bit.apk`

- The original 32-bit game runs on 64-bit-only phones through the ZettaBridge translation layer, with the game APK bundled inside the launcher.

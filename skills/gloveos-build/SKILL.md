---
name: gloveos-build
description: Build, sign, flash and update GloveOS (a GrapheneOS fork for the Pixel 8a with Android 17 Computer Control turned on), and push the Glove priv-app over adb on userdebug builds. Use for syncing the GrapheneOS tag, applying the source overlay, image builds, release signing, factory flashing, OTA sideloads, the app-only dev loop (build-app / dev-install), and staging the on-device speech models. Not for operating the phone (use glove-phone) or changing the harness code itself (use glove-dev).
metadata:
  repo: https://github.com/omeyz/gloveos-skills
---

# Building GloveOS

GloveOS is GrapheneOS at a pinned tag plus a small overlay: Computer Control flag flips, the Glove app as a `system_ext` priv-app signed with the fork's platform key, a framework overlay that makes Glove the default assistant, and two framework patches (double-tap power, super-agent allowlist). Target device: Pixel 8a (`akita`). Everything builds in a Podman container on a Linux host; flashing happens from whichever machine the phone is plugged into.

Read `references/build.md` for the image pipeline and `references/flash.md` for flashing and OTA before running anything.

## Where things are

| Place | What |
|---|---|
| repo `build/scripts/` | every script below; mirror to the host with `rsync -a build/scripts/ fedora:~/glove/scripts/` |
| repo `build/overlay/` | files copied over the GrapheneOS tree (flag values, product makefile, patches) |
| repo `app/` | the Glove app; mirrored to `packages/apps/Glove` in the tree |
| host `~/glove/grapheneos` | the source checkout (hundreds of GB with `out/`) |
| host `~/glove/keys/<device>` | signing keys; they never leave the host and are never committed |
| host `~/glove/third_party/glove-speech` | staged sherpa-onnx libs and speech models (not in git) |
| host `~/glove/grapheneos/releases/<N>/release-akita-<N>/` | factory, OTA and install zips for build number N |

`run.sh <script>` runs a script inside the container with `~/glove` mounted at `/glove`. Scripts marked *host* run outside it.

## The three loops

**Image build and release** (about 10 min incremental plus 5 min signing; 1.5 h cold):

```
bash ~/glove/scripts/image-release.sh 2026110900      # host, detached: overlay → build userdebug → sign + package
tail -f ~/glove/image-release.log
```

`image-release.sh` runs `apply-overlay.sh`, then `BUILD_NUMBER=N run.sh build.sh userdebug`, then `run.sh release.sh userdebug N`. The build number becomes `ro.build.version.incremental`. It refuses to start twice. Do not run `dev-install.sh` while it runs: they share `out/`.

**Update the phone** without wiping:

```
scp fedora:~/glove/grapheneos/releases/N/release-akita-N/akita-ota_update-N.zip releases/N/
build/scripts/flash.sh ota releases/N/akita-ota_update-N.zip   # reboots to sideload, pushes, reboots back
```

**App-only dev loop** (about 1 to 2 min):

```
build/scripts/dev-install.sh            # rsync app + scripts to the host, build Glove, sign with the platform key, install over adb
build/scripts/dev-install.sh --no-build # reinstall the last build
```

GrapheneOS accepts a system-app update only with the same signer, a higher `versionCode` (the script stamps one), the APK's `.idsig`, and the one-time `persist.allow_unknown_system_app_updates=1` (the script sets it; `adb root` works on userdebug). A dev-installed app survives a later OTA as long as its `versionCode` is higher than the image's.

## Checks after a flash or OTA

```
adb shell getprop ro.build.version.incremental                         # the build number you expect
adb shell cmd role get-role-holders android.app.role.ASSISTANT         # app.gloveos.glove
adb shell dumpsys package app.gloveos.glove | grep -c ACCESS_COMPUTER_CONTROL
adb shell ls /system_ext/etc/glove/models/asr /system_ext/etc/glove/models/kws   # speech models present
adb logcat -s GloveRuntime GloveLoop GloveHands GloveRecognizer GloveWake ComputerControlSessionProcessor
```

## Rules

- `flash.sh unlock`, `flash.sh factory` and `flash.sh lock` wipe the phone. Run them only when the user asks for that step in the current conversation.
- Be kind to the host: one build at a time, check `df -h /home` first, never `repo sync` on the host and in the container at once.
- Never print or copy signing keys or the sudo password. The host build runs rootless.
- After changing `app/` or `build/overlay/`, run `apply-overlay.sh` (dev-install.sh and image-release.sh do it for you).

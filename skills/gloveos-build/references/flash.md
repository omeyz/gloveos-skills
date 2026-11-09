# Flashing and updating

Run on the machine with USB to the phone (platform-tools 35.0.1 or newer). `flash.sh` follows https://grapheneos.org/install/cli.

```
build/scripts/flash.sh unlock                                   # once; OEM unlocking must be on; confirm on the phone (wipes)
build/scripts/flash.sh factory releases/N/akita-factory-N.zip   # bootloader, radio, the fork's AVB key, all partitions (wipes)
build/scripts/flash.sh boot                                     # from the bootloader into GloveOS
build/scripts/flash.sh ota releases/N/akita-ota_update-N.zip    # update without wiping
build/scripts/flash.sh lock                                     # M2 only: relock with the fork's key installed (wipes)
```

`flash.sh ota` runs `adb reboot sideload-auto-reboot`, waits for sideload, pushes the zip, and the phone reboots on its own. The dev-installed Glove app survives when its `versionCode` is higher than the image's.

## First-boot checks

```
adb shell getprop ro.build.version.incremental
adb shell cmd role get-role-holders android.app.role.ASSISTANT        # app.gloveos.glove
adb shell dumpsys package app.gloveos.glove | grep -c ACCESS_COMPUTER_CONTROL
adb shell settings get secure voice_recognition_service               # Glove's recognizer, set by the OS
adb shell dumpsys activity services app.gloveos.glove | grep -E "WakeWordService|AgentService"
```

Logcat tags while running a task: `GloveRuntime GloveLoop GloveHands GloveRecognizer GloveWake ComputerControlAllowlistController ComputerControlSessionProcessor`.

## Wipes

`unlock`, `factory` and `lock` erase the phone. Never run them on inference. The user must ask for that step in the current conversation.

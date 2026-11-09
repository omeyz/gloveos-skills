# Image pipeline

Source of truth upstream: https://grapheneos.org/build. GloveOS follows it with scripts in `build/scripts/`.

| # | Script | Where | What |
|---|---|---|---|
| 0 | `host-prep.sh`, `rebuild-image.sh` | host | repo launcher, directories, the `gloveos-build` container image (`build/Containerfile`) |
| 1 | `sync.sh [TAG]` | host | `repo init -b refs/tags/<TAG>`, verify the tag signature, `repo sync -j8`; idempotent and resumable |
| 2 | `apply-overlay.sh` | host | copy `build/overlay/**` into the tree, apply `build/patches`, mirror `app/` to `packages/apps/Glove`, mirror the staged speech files into `packages/apps/Glove/speech/` (warns if not staged) |
| 3 | `run.sh vendor.sh` | container | adevtool: build its host tools, fetch the Pixel factory image, extract vendor files, generate the device tree |
| 4 | `run.sh build.sh userdebug` | container | `lunch akita-cur-userdebug; m vendorbootimage vendorkernelbootimage target-files-package`. `BUILD_NUMBER` in the environment sets `ro.build.version.incremental` (default: today's `YYYYMMDD00`) |
| 5 | `run.sh keys.sh` | container, once | release keys (`make_key` set, `avb.pem`, `avb_pkmd.bin`, OpenSSH key). Dev keys have no passphrase; `GLOVE_KEYS_PASSPHRASE` encrypts them |
| 6 | `run.sh release.sh userdebug N` | container | `m otatools-package; script/finalize.sh; script/generate-release.sh akita N` → `releases/N/release-akita-N/akita-{factory,ota_update,install}-N.zip` |
| 7 | `run.sh build-app.sh` | container | `m Glove GloveOSFrameworkOverlay apksigner`, re-sign with the platform key → `~/glove/out-app/Glove.apk` |
| – | `image-release.sh [N]` | host | detached chain of 2 + 4 + 6 with a log at `~/glove/image-release.log`; refuses a second instance |
| – | `fetch-speech.sh` | host | stage sherpa-onnx v2.0.1 Android libs plus the English streaming zipformer2 ASR model and the "Hey Glove" keyword-spotting model into `~/glove/third_party/glove-speech/` |
| – | `pipeline.sh`, `start-pipeline.sh`, `bg.sh` | host | unattended sync → overlay → vendor → build; a detached runner with pidfile and log |

## Variants

- `userdebug` (M1, "see it move"): dev keys, bootloader unlocked, `adb root` works, Computer Control's super-agent allowlist honoured.
- `user` (M2): production variant. Upstream honours `config_computerControlKnownSuperAgents` only on debuggable builds, so the user build needs the framework patch in `build/patches` or first-boot DeviceConfig allowlists.

## Speech models

The Glove app bundles sherpa-onnx (`jni_libs` in `app/Android.bp`) and ships the models on the system image:

- ASR: streaming zipformer2 English int8 (2023-06-26), `/system_ext/etc/glove/models/asr`
- KWS: gigaspeech 3.3M keyword spotter with `keywords.txt` (`HEY_GLOVE`, `OK_GLOVE`), `/system_ext/etc/glove/models/kws`

Both are `prebuilt_etc` modules declared in `app/Android.bp` and sourced from `packages/apps/Glove/speech/`. The app falls back to `<files dir>/models/{asr,kws}` when the system copies are missing, which is how models are tried on the phone before an image build (push them with adb as root and `chown` to the app uid).

Gotcha: `modelType` must be `zipformer2` for these models. With `zipformer` the engine exits the process with no crash log.

## Timings and disk (Framework Desktop, 32 threads)

- Cold image build: about 1 h 25 min. Incremental: about 10 min. Signing and packaging: about 5 min.
- App-only build: 1 to 2 min.
- Plan on 250 GB for source + out + ccache. Check `df -h /home` before each build.

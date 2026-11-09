---
name: glove-dev
description: Change the Glove harness app (Kotlin, app/glove in the GloveOS repo): add or edit agent tools (Tools.kt and AgentLoop.kt), the approval gate and permission modes, the observe/act loop, the model wire formats (Brain.kt, Anthropic Messages and OpenAI chat completions), the Computer Control session wrapper (Hands.kt, Observation.kt), the voice path (VoiceInteractionService, RecognitionService, wake word) and the sherpa-onnx speech engine. Use when adding a structured tool with its risk class and portable schema, changing what needs approval, or debugging a run from logcat. Not for building or flashing the OS image (gloveos-build) or for driving the phone (glove-phone).
metadata:
  repo: https://github.com/omeyz/gloveos-skills
---

# Changing the Glove harness

Glove is one priv-app (`app.gloveos.glove`) that holds the assistant role, opens a Computer Control session on a hidden virtual display, and runs an observe/act loop against the model the owner configured. Read `references/tool-conventions.md` before adding a tool.

## File map (`app/glove/src/main/java/app/gloveos/glove/`)

| File | Owns |
|---|---|
| `Tools.kt` | tool schemas, OpenAI function format (`Brain.kt` converts for Anthropic). `selectApps` for phase A, `act` for phase B |
| `AgentLoop.kt` | the loop: `selectApps` → session → `act`. `execute()` runs a call, `describe()` writes the transcript line, `approvalFor()` is the mode gate, the system prompt is in `act()` |
| `Brain.kt` | `AnthropicBrain` and `OpenAiBrain`: history, images, tool results, `trim()` |
| `Hands.kt` | the Computer Control session: launch, tap, long-press, swipe, insertText, back, windows, screenshot, hand over |
| `Observation.kt` | accessibility tree → numbered element list (160 cap) with labels used by the gate |
| `WebReader.kt` | `web_read`: HTTP GET → readable text, labelled untrusted |
| `Settings.kt`, `SettingsActivity.kt` | provider, model, effort, mode (MANUAL / AUTO / FULL), blocked apps, speech, limits; the Glove page in system Settings |
| `GloveRuntime.kt`, `AgentService.kt` | task lifecycle, foreground service, stop |
| `voice/` | `GloveVoiceInteractionService`, `GloveSession` (aura overlay, mirror, approvals, transcript), `GloveRecognitionService`, `WakeWordService` |
| `speech/SpeechEngine.kt` | sherpa-onnx recognizer and keyword spotter, model lookup, `Mic` with AGC |
| `ui/` | overlay views |

## Add a tool

1. **Schema** in `Tools.act`: `tool(name, description, properties, required)`. Name `[a-z0-9_]`, domain-prefixed for structured tools (`sms_send`, `alarm_set`), bare verbs for UI actions. Description of 3 to 4 sentences: what, when, when not, caveats. `additionalProperties: false` is added for you.
2. **Execution** in `AgentLoop.execute()`: a `when` branch returning `Outcome(resultForModel, lineForUser?, finished?, wantsScreenshot?)`. Failures return `"error: <what to do next>"` for now (an `is_error` tool result is planned).
3. **Transcript line** in `describe()`: one plain-language line, or null if the tool reports itself.
4. **Risk class** in `approvalFor()`. Today: `read` tools run in every mode without a prompt; UI actions are MANUAL-confirmed; AUTO confirms the `SENSITIVE` label class; sends and calls (`isSendOrCall`) confirm in every mode including FULL (owner decision 2026-11-09). A tool registry with explicit classes replaces these regexes next.
5. **Prompt**: one sentence in the `act()` system prompt if the model needs to know when to prefer the tool.
6. **Permissions**: declare in `AndroidManifest.xml`; priv-app permissions need the privapp allowlist in `build/overlay`. The app is platform-signed, so `signature` permissions are reachable.
7. **Build and try**: `build/scripts/dev-install.sh`, then run a goal on the phone and watch `adb logcat -s GloveLoop GloveHands GloveRuntime`.

## Conventions

- Keep `tools` and the system prompt byte-identical for a whole run (prompt caching, preserved thinking). Gate a tool by returning an error, not by removing it mid-run.
- Element ids are valid only for the latest reading. Every action re-captures; tell the model to re-read after each step.
- Anthropic path: images go in the user turn after the tool results (`userImage`); `max_tokens` and caching live in `AnthropicBrain.chat`.
- Speech: `SpeechEngine` looks in `/system_ext/etc/glove/models/{asr,kws}` and falls back to the app's files dir. `modelType` is `zipformer2`.
- Voice entry: power button or "Hey Glove" → `GloveSession.onShow` → auto-listen → goal → `GloveRuntime`.

## Debugging

```
adb logcat -s GloveLoop GloveHands GloveRuntime GloveRecognizer GloveWake GloveSpeech ComputerControlSessionProcessor
adb shell dumpsys activity services app.gloveos.glove
```

A silent process exit right after model load is almost always a wrong `modelType`. An approval prompt that did not appear is `approvalFor()` returning null: check the element label and the mode.

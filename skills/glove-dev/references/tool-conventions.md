# Tool conventions (internal loop and MCP)

Target: MCP spec 2026-07-28, Anthropic Messages API, OpenAI chat completions. One registry should generate all three.

## Names and schemas

- Names: `[a-z0-9_]`, at most 64 characters. Valid for MCP, Anthropic (`^[a-zA-Z0-9_-]{1,128}$`) and OpenAI (no dots). Clients prefix (`mcp__glove__tap`), so keep UI verbs bare and prefix structured tools by domain (`sms_send`, `calendar_list`).
- Descriptions: 3 to 4 sentences: what it does, when to use it, when not to, caveats. Add input examples where the target is a union.
- Schemas: `type: object`, `additionalProperties: false`; `{type: object, additionalProperties: false}` for no parameters. Avoid root `oneOf` (local OpenAI-compatible servers mishandle it); use optional fields and validate in the harness.

## Results

- MCP: `content` plus `structuredContent` (same JSON in a text block for old clients) plus `outputSchema`. Execution failures: `isError: true` with a message that says what to do next. Unknown tools and malformed calls are protocol errors.
- Anthropic: `is_error` on the `tool_result`; put instruction text before the image in a `screenshot` result.
- OpenAI chat completions: tool messages are text only; the follow-up user turn carries the image.

## Annotations (MCP) and Glove's risk classes

Defaults: `readOnlyHint` false, `destructiveHint` true, `idempotentHint` false, `openWorldHint` true. Hints are untrusted by clients; the phone enforces modes itself. Put Glove's class in `_meta` as `"app.gloveos/risk"`.

| Class | Manual | Auto | Full | Examples |
|---|---|---|---|---|
| read | runs | runs | runs | `device_state`, `apps_list`, `screen_read`, `screenshot`, `web_read` |
| sensitive-read | runs, tainted, logged | same | same | `contacts_search`, `sms_list`, `notifications_list`, `location_get` |
| local-write | confirm | runs | runs | `alarm_set`, `setting_set`, `notification_dismiss` |
| send | confirm | confirm | confirm | `sms_send`, `call_place`, UI taps on Send/Call/Dial (owner decision 2026-11-09) |
| destructive | confirm, logged | runs, logged | runs, logged | `file_write`, deletes |
| hard | confirm | confirm | runs | payments, policy changes |

## State

No protocol sessions in 2026-07-28. Return handles as arguments: `session_id` from `session_start`, `snapshot_id` from every observation. Element refs are valid only with their snapshot; reject stale ones with an actionable error and stop the rest of the turn.

## Determinism

List tools in a fixed order. Keep `tools` and `system` byte-identical for a run. Gate by error, never by removing a tool mid-run.

## Planned UI tool surface

`session_start {apps[], goal?}`, `app_open {package}`, `tap {element?, snapshot_id?, x?, y?}`, `long_press {same}`, `scroll {direction, element?, amount?}`, `swipe {x1,y1,x2,y2,duration_ms?}`, `type_text {text, element?, replace?, submit?}`, `press_key {key}`, `wait {seconds, until_text?}`, `screen_read {filter?, max_elements?}`, `screenshot {region?}`, `hand_over {note}`, `finish {status, summary, evidence}`. Action results: `{ok, changed, snapshot_id, package, keyboard_visible, screen}`.

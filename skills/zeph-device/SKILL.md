---
name: zeph-device
description: Control a Zeph device through the companion's MCP server - read device state, change settings, push screens to the device, receive button intents, and register your own device app. Use when a task involves a Zeph device, glances, pushed device UI, or device-button round trips.
metadata:
  zeph:
    device: true
    version: 1.1.0
    capabilities: ["zui.interactive", "notify.reply"]
---

# zeph-device — the base skill for Zeph's MCP server

The Zeph companion (the desktop app) runs a localhost MCP server. Connecting
to it makes you a device controller and, optionally, the host part of a device
app. This skill teaches the protocol surface, the pushed-UI tree format, and
the recipes that work well on a 1.8" device screen.

## 1. Connecting

The user mints your access token in the companion's **MCP** page (Copy puts a
ready snippet on the clipboard, e.g.):

```sh
claude mcp add zeph --transport http http://127.0.0.1:<port>/mcp \
  --header "Authorization: Bearer zmcp_..."
```

- Transport is **streamable HTTP**; every request carries the Bearer token.
- The first time you connect, the user must **approve** you in the companion
  (a consent dialog). Until then tool calls return "awaiting approval" — tell
  the user to open the Zeph app and approve.
- Scopes: `control` (device control, tier 1) and `apps` (register device apps,
  tier 2). Your tools/list shows only what your token grants.

## 2. Capabilities (tools)

| Tool | Scope | What it does |
|---|---|---|
| `list_devices` | control | Every known device: serial, name, state, transports, battery. |
| `get_device_state` | control | One device + its device-synced settings (brightness, device_name, volume, `mic_gain` [labeled "Mic level"], `mic_agc`, `storage_backend`). |
| `update_settings` | control | Write device-synced settings; per-setting outcome (synced / offline / validation error). |
| `push_screen` | control | Push an ephemeral zui tree to a snapshot ref; optional `ttl_seconds` auto-clear. |
| `notify` | control | Post on the device's notification ladder (badge -> modal). Prompt posts WAIT and return the user's answer. No app setup needed. |
| `subscribe_intents` | control | Device button intents stream to you as `notifications/message` (logger `zeph.intents`). |
| `register_app` | apps | Register your app manifest (exports + glances) on the device + companion library. |
| `update_snapshots` | apps | Push live content for your app's snapshot-backed exports. |
| `remove_app` | apps | Evict your app from devices + the library. |

Device state is also readable as resources: `zeph://devices`,
`zeph://device/{serial}`.

## 3. The pushed-UI tree (zui JSON IR)

Screens are JSON node trees. Every node is an object whose first key is `k`
(the kind); children go in `c`. Text-bearing nodes use `t`.

**The full per-kind registry lives in
[references/widgets.md](references/widgets.md)** - all 36 kinds (incl. the
`pageview`/`choice` modal-doc kinds + the `image` asset kind), every field that actually decodes on
the device, a working example per kind, and an explicit streamable yes/no
flag. Load it before composing anything beyond the recipes below. As of #328
every content kind streams (chips/tabs/picker/select/dial/clock/widget/
stepper via `opts`/`on`/`sel`/... keys; applist takes children); the one
exception is `slot` (glance-composition only - never push it).

Common keys: `t` text · `ic` icon name · `sub` subtitle · `id` id ·
`v` value (uint) · `lyt` layout (1 = centered) · `cv` row chevron ·
`h` gap height px · `c` children.

**Interactive attrs (wire v12/v13 — the v12 role/key/`lv` set + the v13 #358 ring/grid/row attrs; interactive value controls report on s6 `0x06` per widgets.md):**

| Attr | On | Values | Meaning |
|---|---|---|---|
| `role` | `btn` | `submit` \| `dismiss` \| `event` \| `next` \| `prev` | `submit`/`dismiss` are TERMINAL (the modal resolves and your blocked `notify` call returns `{action, values, steps}`); `event` pings you and the modal stays (abort/progress); `next`/`prev` page locally, never reported |
| `key` | `btn` | `up`, `down`, `up.double`, `down.double` | bind a physical button — the button renders the key's glyph; hold is never bindable (push-to-talk floor) |
| `live` | `slider` `segslider` `toggle` `select` | `true` | stream value changes to you as widget events (debounced) instead of waiting for the terminal reply (`select` reports the tapped option index) |
| `advance` | `choice` | `true` | wizard auto-advance: picking an option acks and pages forward; terminal on the last page (the picked option id becomes the `action`) |

Every value widget (`slider`/`segslider`/`toggle`/`choice`) carrying an `id`
collects into the terminal `values{}` map — nothing leaves the device until a
terminal action. Button labels: one line, max 24 ASCII chars (18 when
`key`-bound — the glyph reserves width); longer labels are rejected with the
node id. Re-posting `notify` with a LIVE `id` updates that notification
in place (no re-surface flash) — the progress/monitor pattern.

A face/full-screen push:

```json
{ "k": "screen", "lyt": 1, "c": [
  { "k": "eyebrow", "t": "BUILD" },
  { "k": "big", "t": "Green" },
  { "k": "gap", "h": 8 },
  { "k": "text", "t": "47 tests passed" }
] }
```

A swipe-up page (list content):

```json
{ "k": "list", "c": [
  { "k": "row", "t": "zeph", "ic": "folder", "cv": 1 },
  { "k": "row", "t": "companion", "ic": "folder", "cv": 1 }
] }
```

A widget slot (cell/hero family):

```json
{ "k": "box", "c": [
  { "k": "eyebrow", "t": "5H LIMIT" },
  { "k": "big", "t": "64%" }
] }
```

## 4. Recipe: ephemeral screen + button reply (the round trip)

1. `list_devices` → pick a `serial`.
2. `subscribe_intents { serial }` — keep the session open; intents arrive as
   notifications with data `{serial, id, phase}` (`press`/`release`/`tap`/
   `double`; press+release pairs = hold).
3. `push_screen { serial, ref, tree, ttl_seconds }` — `ref` must be a
   snapshot-backed export of a registered app (`app_id.export_id`). Content
   shows wherever that export sits on a glance.
4. The user presses the bound button → you get the intent → push the next
   screen. That's the whole loop.

Dismiss semantics: a push REPLACES the export's previous content and stays
until the next push or reboot; `ttl_seconds` makes the companion push a blank
tree afterwards. Nothing navigates — you only ever change content.

## 4b. Recipe: notifications (the `notify` verb)

Notifications need ZERO setup — no app, no glance. One tool, six rungs;
content richness scales with the level:

| Level | Surface on the device | Use it for |
|---|---|---|
| `badge` | statusbar bell + unread count | "something happened, look later" |
| `marquee` | one live status line at the top | progress that keeps changing ("build 2/3") |
| `notice` | toast card (icon + title + text), auto-dismisses | FYI moments ("tests green") |
| `urgent` | fullscreen takeover, may wake the screen | needs eyes NOW (requires the `urgent` scope) |
| `prompt` | question + up to 3 options on the physical buttons | ask-me decisions (the `AskUserQuestion` shape) |
| `modal` | multi-page zui document (`body` arg) | rich review/confirm flows (paged, with buttons) |

Retention is opt-in (`persist`, default false): a post renders its rung,
then vanishes on resolution (dismissed / answered / expired) — no entry in
the device's notification list, no bell count. Pass `"persist": true` when
it should stay as unread history (the bell counts persisted-unread only —
a `badge` post without it is invisible).

Per-rung recipes:

```json
{ "serial": "...", "title": "Tests green", "text": "412 passed", "icon": "check" }
```
(default level = notice — fire and forget.)

```json
{ "serial": "...", "level": "prompt", "title": "Deploy to production?",
  "options": ["Deploy", "Hold"], "ttl_seconds": 60 }
```
The call BLOCKS until the user answers (options auto-bind to the device's
physical buttons and render with the matching glyphs) or the TTL resolves
`default_option`. The result carries `reply: {option, label}` — or
`"dismissed"` if the user swiped it away.

```json
{ "serial": "...", "level": "marquee", "title": "build 2/3 - linking",
  "color": "info" }
```
A newer marquee post replaces the line; it expires on its TTL (default
30 s). `color` tints the strip: a palette token (`info`/`success`/`warn`/
`error` or the vibrant set `blue`/`teal`/`green`/`amber`/`orange`/
`crimson`/`iris`/`violet`) or `#RRGGBB` (snapped to the nearest hue).

Modal — a multi-step document (AskUserQuestion-class flows). `body` is a zui
JSON tree; a root `pageview` makes the pages (the device frames them with a
STEP n OF m eyebrow + paging dots — don't add your own). The call BLOCKS
like a prompt and returns the TERMINAL reply
`reply: {action, values, steps}`:

```json
{ "serial": "...", "level": "modal", "title": "Review deploy",
  "body": { "k": "pageview", "c": [
    { "k": "box", "c": [
      { "k": "text", "t": "Pick an environment" },
      { "k": "choice", "id": "env", "c": [
        { "k": "text", "id": "staging", "t": "Staging" },
        { "k": "text", "id": "prod", "t": "Production" } ] } ] },
    { "k": "box", "c": [ { "k": "text", "t": "14 commits - rollback ready" } ] },
    { "k": "box", "c": [
      { "k": "btn", "id": "confirm_deploy", "t": "Deploy",
        "role": "submit", "key": "up" } ] } ] } }
```

The user pages with the footer chevrons, a horizontal swipe, or the physical
buttons (up = previous, down = next, up double-tap = dismiss — unless a page
element `key`-binds them). Picks and widget values collect device-side; the
`submit` press resolves the call with
`{action: "confirm_deploy", values: {env: "prod"}, steps: [...]}`. An
invalid or oversized `body` downgrades modal -> prompt.

**Recipe: wizard (auto-advance).** Each page is a `choice` with
`advance: true` — a pick acks, collects, and pages forward; the LAST page's
pick is terminal and its option id becomes the `action`. One call returns
the whole path:

```json
{ "serial": "...", "level": "modal", "title": "Deploy",
  "body": { "k": "pageview", "c": [
    { "k": "box", "c": [
      { "k": "text", "t": "Pick an environment" },
      { "k": "choice", "id": "env", "advance": true, "c": [
        { "k": "text", "id": "staging", "t": "Staging" },
        { "k": "text", "id": "prod", "t": "Production" } ] } ] },
    { "k": "box", "c": [
      { "k": "text", "t": "Pick a region" },
      { "k": "choice", "id": "region", "advance": true, "c": [
        { "k": "text", "id": "eu", "t": "Europe" },
        { "k": "text", "id": "us", "t": "US" } ] } ] },
    { "k": "box", "c": [
      { "k": "text", "t": "Ready to deploy" },
      { "k": "btn", "id": "go", "t": "Deploy", "role": "submit",
        "key": "up" } ] } ] } }
```
Returns `reply: {action: "go", values: {env: "...", region: "..."}}`.

**Recipe: live control.** A `live: true` widget streams every change to you as
widget events on the intents notification channel while the modal stays up;
`Done` closes it with the final values:

```json
{ "serial": "...", "level": "modal", "title": "Volume",
  "body": { "k": "box", "c": [
    { "k": "text", "t": "Drag to adjust" },
    { "k": "slider", "id": "level", "v": 40, "live": true },
    { "k": "btn", "id": "done", "t": "Done", "role": "submit",
      "key": "up" } ] } }
```
Widget events arrive as `{id, event: "widget", wid: "level", value: n}` —
apply them host-side as they stream (debounced device-side).

**Recipe: monitor + abort.** Post once, then RE-POST with the returned `id`
to update in place (no flash); a `role: "event"` button pings you without
closing; finish by updating to a terminal state or dismissing over the rail:

```json
{ "serial": "...", "level": "modal", "title": "Deploying",
  "body": { "k": "box", "c": [
    { "k": "text", "t": "Step 2 of 5 - migrations" },
    { "k": "progress", "v": 40 },
    { "k": "btn", "id": "abort", "t": "Abort", "role": "event" } ] } }
```
Then every few seconds: the same call + `"id": <returned id>` with a fresh
`text`/`body`. An `abort` press arrives as a widget event
(`wid: "abort", value: 1`) — stop the job, then update the body to the
final state with a `role: "submit"` Done button.

**Downgrade table** — the device may lower the rung; the result reports the
effective `level` (+ `downgraded_from`):

| You asked for | You may get | When |
|---|---|---|
| `modal` | `prompt` | body missing/invalid/too big |
| `prompt` | `notice` | no options given |
| `urgent` | `notice` | your token lacks the `urgent` scope |
| anything | `badge` | Do Not Disturb is on (unless the post is critical-class) |
| (dropped) | error `EACCESSDENIED` | the user muted your source in the companion |

Etiquette: your client name is the `source` the user sees (and mutes!) —
batch low-value updates into a marquee or badge instead of notice spam.
Dismissals/replies also stream on the intents notification channel keyed by
the returned `id`, so you can observe resolution without blocking.

## 5. Recipe: your own device app (tier 2)

`register_app` with a manifest (AppPackage JSON):

```json
{ "manifest": {
  "id": "build_bot",
  "name": "Build Bot",
  "icon": "bot",
  "description": "CI status on your Zeph.",
  "pages":   [ { "id": "log", "title": "Last build", "ir": null } ],
  "widgets": [ { "id": "status", "family": "hero",
                 "placeholder": { "k": "box", "c": [ { "k": "eyebrow", "t": "BUILD" }, { "k": "big", "t": "--" } ] } } ],
  "intents": [ { "id": "rerun", "label": "Re-run build", "kind": "host", "device_action": 0 } ],
  "glances": [ { "id": "main", "title": "Build Bot",
                 "cfg": { "id": "build", "face": { "kind": "template", "ref": "tpl.hero" },
                          "slots": [ { "idx": 0, "ref": "build_bot.status" } ],
                          "pages": [ "build_bot.log" ],
                          "bindings": [ { "button": "main", "gesture": "tap", "intent": "build_bot.rerun" } ],
                          "class": "tailored", "source_ref": "build_bot.main", "options": [] } } ]
} }
```

- `ir: null` pages and all widgets are snapshot-backed: feed them with
  `update_snapshots { app_id, snapshots: { "build_bot.status": {...} } }`.
  Refs must carry YOUR app's prefix.
- The user adds your glance from the companion's glance editor (it appears in
  the gallery immediately) — or you push to exports already placed.
- Your `host`/`composite` intents come back to you via `subscribe_intents`
  as `build_bot.rerun` etc.
- If you disconnect, your last content stays on the device and goes stale —
  re-register + re-push on reconnect. `remove_app` evicts everything.

## 6. Best practices (1.8" screen, 368x448)

- **One idea per screen**: eyebrow (context, UPPERCASE, short) + big (the
  payload, 1-3 words) + at most one supporting text line.
- Text must be glanceable: no paragraphs, no markdown, ~16 chars per line.
- Use `ttl_seconds` for transient notices (30-120 s) so the surface cleans
  itself; long-lived state belongs in a registered app's widget, not a
  pushed one-off.
- Push on CHANGE, not on a timer — snapshots are cached on-device; identical
  re-pushes waste the link.
- Hold semantics: act on `press`, finish on `release`; `tap` for one-shots.
- Don't poll `get_device_state` in a loop; subscribe to intents and read
  state when you actually need it.
- Battery/connection honesty: `list_devices` tells you if the device is
  offline — queue content and push after reconnect instead of erroring.

## 7. Troubleshooting

- `401` — token missing/revoked: ask the user for a fresh snippet from the
  MCP page.
- "awaiting approval" — the user must approve you in the companion (MCP page).
- "does not carry the 'apps' scope" — the token was minted without
  "Allow installing apps"; the user can mint a new one.
- `push_screen` says offline — the device isn't connected; check
  `list_devices`.
- Pushed content not visible — the `ref` must be an export of a REGISTERED
  app and that export must sit on a glance the user can see.

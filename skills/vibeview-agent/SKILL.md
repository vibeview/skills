---
name: vibeview-agent
description: Verify a React Native change on a real running app — install/launch it on a live iOS, Android, Apple TV or Android TV device via VibeView, drive the UI, and check the result. Use when asked to verify this change on a device, test my React Native app, or run it on a simulator.
---

# VibeView agent control

Drive a live iOS/Android VibeView session from the CLI to verify a React
Native change actually works, not just that it compiles. Requires the
`vibeview` CLI installed and authenticated (`vibeview login`, or
`VIBEVIEW_API_TOKEN` set).

## 1. Build requirements

Read before uploading anything — a build that violates these will install
and then crash or refuse to boot:

- **Android**: the debug APK must include an `x86_64` slice. Standard
  `./gradlew assembleDebug` output includes it by default; only an issue if
  the project filters ABIs.
- **iOS**: upload a **simulator** debug build (not a device `.ipa`), and it
  must include the `arm64` slice.

Debug builds are auto-detected on upload — no separate flag needed. Upload
warnings are the safety net: if a build is missing the right architecture,
`upload-app` says so.

Full detail: https://vibeview.io/docs/live-development and
https://vibeview.io/docs/preparing-your-build.

## 2. The loop

1. Upload the debug build — **only after a native change** (new/updated
   native module, Podfile/Gradle change). Everyday JS/asset edits never need
   a re-upload:
   ```bash
   vibeview upload-app ./path/to/app-debug.apk
   ```
2. Start Metro in the React Native project (`yarn start` or equivalent) and
   leave it running.
3. Start a dev session in the background and get structured output:
   ```bash
   vibeview dev --detach --json
   ```
4. Parse the `session_ready` line from stdout:
   ```json
   {"event":"session_ready","session_id":"...","url":"...","pid":1234}
   ```
   Use `session_id` as `--session <id>` on every verb below. The event
   fires once the session accepts commands (device up, app installed) —
   no need to poll `dev-status` first. If the app ships no native changes
   since the last run, skip step 1 and just do steps 2–4.
5. **Immediately tell the developer the session `url`** — "watch or take
   over at any time: `<url>`". The page is a live, fully interactive view
   of the device; the developer watching along is a feature, not a risk.
   (For a session you attached to with `--session` instead of starting,
   the page is `https://vibeview.io/sandbox/<session_id>`.)

Running `dev` writes two things into the project root: a `.vibeview/`
directory (live session state — local machine state, never commit) and a
`vibeview.json` (your platform/app/build choices, which teammates DO want).
Make sure `.vibeview/` is in the project's `.gitignore`, adding it if
missing. Leave `vibeview.json` tracked, but tell the developer you created
it rather than letting it appear unexplained in their `git status`.

## 3. Verifying

- **Always run `ui-tree` first** — every ref (`@e5`) the UI exposes comes
  from here. Never guess a ref.
  ```bash
  vibeview ui-tree --session <id>
  ```
  An empty or near-empty first tree usually means the app is still booting
  (splash screen, fetching remote config) — NOT a broken session. Take a
  screenshot to confirm, `wait`, and fetch the tree again before concluding
  anything is wrong.
- Act on the screen using refs from the tree:
  ```bash
  vibeview tap @e5 --session <id>
  vibeview type "hello@example.com" --session <id>
  vibeview scroll down --session <id>
  vibeview press back --session <id>
  ```
- **Read the observation in every response before deciding the next
  action.** Every mutating verb returns one of: screen unchanged, a
  `+`/`~`/`-` diff with fresh `@e` refs, a full tree, or an "unavailable"
  warning. Refs go stale the moment the screen changes — a stale ref is
  refused. **The refusal does NOT include the current screen**: because the
  action never ran, the observation comes back as "screen unchanged" with no
  tree, so run `ui-tree` again to get live refs before retrying. Never reuse
  a ref from a previous observation.
- **Take a screenshot whenever you changed how something LOOKS.** The UI
  tree proves an element exists, has the right text, and occupies the right
  rect — it says nothing about colour, contrast, or whether something is
  hidden behind another view, so a screen can pass every check the tree can
  express and still be visibly broken (text the same colour as its
  background, a view drawn on top of another). Tree for structure and refs;
  screenshot for appearance.

  **Actually look at the image** — confirming the file was written proves
  nothing about how it renders.
  ```bash
  vibeview screenshot --out ./check.png --session <id>   # then open it
  ```
  Over MCP, `screenshot` only writes the file and returns its path unless
  you pass `inline: true`, which returns the image itself. If you cannot
  open files on the machine running the server, `inline: true` is the only
  way you will ever see the screen. It costs a lot of context, so use it
  when appearance matters — not to confirm a tap landed, which the
  response's observation already tells you.
- **When something breaks — a crash, a frozen screen, an unexpectedly empty
  tree — read the app logs before guessing.** `logs` returns the app's own
  output from the device: JS console lines, native errors, and crash
  messages (not full system logs, and not Metro's terminal — native crashes
  never appear there). The response ends with a `cursor`; pass it back as
  `--since` to see only lines that arrived after your last read, which is
  the natural loop for "did my action log what I expected".
  ```bash
  vibeview logs --session <id>                 # recent lines (default 100)
  vibeview logs --since 42 --session <id>      # only lines after cursor 42
  ```
- **Off-screen elements**: use `scroll-to <text>` rather than repeating
  `scroll down`. It scrolls until an element whose label, value, or
  accessibility id contains that text is visible, and stops early when the
  content stops changing (end of list). Omit `--direction` for vertical
  lists (it auto-detects); pass it explicitly for horizontal strips.
  ```bash
  vibeview scroll-to "Delete account" --session <id>
  ```
- **Precise two-point gestures** (sliders, drag-and-drop reorder, map
  panning) need `drag` — `swipe`/`scroll` are direction-only and can't
  express them. Each end is independently a ref or `x,y`:
  ```bash
  vibeview drag @e5 @e9 --session <id>
  vibeview drag 100,400 300,400 --session <id>
  vibeview drag @e5 300,400 --press-ms 500 --session <id>
  ```
  Use `--press-ms 500` or more for long-press-to-reorder lists, and
  `--velocity 150` for slow, precise slider adjustments.
- **iOS system alerts** (permission prompts and similar) sit above the app
  and block it. Call `alert get` first to read the message and the exact
  button labels, then act on one:
  ```bash
  vibeview alert get --session <id>
  vibeview alert accept --button "Allow" --session <id>
  ```
- Use `open-url` to jump straight to a deep link instead of navigating by
  hand:
  ```bash
  vibeview open-url "myapp://settings" --session <id>
  ```
- **TV apps**: start the session with `--platform tvos` (Apple TV) or
  `--platform androidtv` (Android TV). Navigate with the d-pad via `press`
  (`dpad_up`/`dpad_down`/`dpad_left`/`dpad_right`/`dpad_center`), or move
  focus directly with `tap-focused` (moves focus AND activates) / `focus`
  (moves focus only, no activation).

## 4. When only a human can act — stop and hand off

Some screens need things you must never guess or invent: **login
credentials, 2FA/OTP codes, CAPTCHAs, destructive confirmations
("Delete account?"), anything payment- or real-account-shaped.** When you
hit one:

1. **Stop driving.** Do not type guessed credentials, do not tap through a
   destructive confirmation to "see what happens".
2. **Ask the developer, naming exactly what you need and where**: "I'm at
   the login screen — please sign in at `<session url>` and tell me when
   you're done." The session page is fully interactive; the developer can
   drive the device directly while you wait.
3. **On resume, run `ui-tree` FIRST** — the human just changed the screen,
   so every ref you held is stale — and **verify the blocker is actually
   gone from the tree** (e.g. the login form is absent) instead of trusting
   the confirmation alone. If it's still there, say what you still see and
   ask again.

## 5. Cleanup

**Always** stop the session when done verifying — a running session bills
streaming minutes even when idle:
```bash
vibeview dev-stop
```

## 6. Command reference

Every verb accepts `--session <id>` to target a specific session (falls
back to the current project's `vibeview dev --detach` session if omitted)
and `--json` to print the raw response envelope as one JSON line instead of
human-readable text.

| Verb | Args | Purpose |
|------|------|---------|
| `ui-tree` | — | Fetch the ref-annotated UI tree for the current screen. |
| `logs` | `[--tail <n>] [--since <cursor>]` | Read recent app logs from the device (JS console, native errors, crash messages). The response's `cursor` feeds the next call's `--since`. |
| `screenshot` | `[--out <path>]` | Capture a screenshot of the current screen and save it to disk. |
| `tap` | `<target>` | Tap an element by ref (e.g. `@e5`), or raw coordinates (e.g. `100,200`). |
| `long-press` | `<ref>` `[--ms <n>]` | Long-press (touch and hold) an element by its ref. |
| `swipe` | `<direction>` `[--distance <short\|medium\|long>]` | Perform a scrollbar-semantic swipe gesture in one of four directions. |
| `scroll` | `<direction>` | Scroll the current view up, down, left, or right using a gesture preset. |
| `scroll-to` | `<text>` `[--direction <up\|down\|left\|right>] [--max-scrolls <n>] [--element-type <type>]` | Scroll until an element matching the text is visible. Stops at end of list. |
| `drag` | `<from>` `<to>` `[--velocity <n>] [--press-ms <n>] [--hold-ms <n>]` | Drag between two points, each an `@ref` or `x,y`. The only precise two-point gesture. |
| `alert` | `<get\|accept\|dismiss>` `[--button <label>]` | Inspect or respond to a visible iOS system alert. |
| `type` | `<text>` | Type text into the currently focused input field. |
| `clear-text` | — | Clear the text in the currently focused input field. |
| `press` | `<button>` | Press a device button or perform a system gesture (home, back, d-pad, etc). |
| `open-url` | `<url>` | Open a deep link or URL in the app under test. |
| `wait` | `[--ms <n>]` | Wait for a specified number of seconds before continuing (default: 2s). |
| `find` | `<text>` `[--below <text>] [--above <text>] [--near <text>]` | Find an element by text with optional spatial constraints. |
| `tap-focused` | `<ref>` | TV: move focus to an element and press SELECT in one step. |
| `focus` | `<ref>` | TV: move focus to an element without activating it. |

Dev-loop lifecycle commands (not registry verbs, but needed for every run):

| Command | Purpose |
|---------|---------|
| `dev --detach --json` | Start a device + Metro-tunnel session in the background, emitting `session_ready`/`warning`/`error` JSON events on stdout. |
| `dev-status` | Check on the current project's detached dev session. |
| `dev-stop` | End the current project's detached dev session. |

## 7. If you're calling VibeView over MCP

Everything above is also exposed as MCP tools by `vibeview mcp`, so an agent
with no shell can run the whole loop. Tool names are the CLI verbs with
hyphens replaced by underscores (`ui_tree`, `tap`, `scroll_to`, `long_press`,
`clear_text`, `open_url`, `tap_focused`, ...), each taking its args by name
(`tap {ref: "e5"}` or `tap {target: "100,200"}`, `scroll_to {text: "..."}`,
`drag {from: "@e5", to: "300,400"}`, `alert {action: "get"}`) plus an
optional `session_id`.

Three tools exist **only** over MCP, with no registry verb — they cover the
steps the CLI does with plain shell commands:

| MCP tool | Args | Purpose |
|----------|------|---------|
| `upload_app` | `file` (required path), `name` | Upload a debug build. The MCP equivalent of `vibeview upload-app`, so step 1 of the loop needs no shell. Returns the `app_id` to pass to `dev_start`. |
| `dev_start` | `platform` (required: `ios`/`android`/`tvos`/`androidtv`), `app`, `metro_port` | Start a dev-loop session held open by the MCP server. The MCP equivalent of `vibeview dev --detach`. Returns `page_url` — relay it to the developer immediately, same as step 5 of the loop. |
| `dev_stop` | — | Stop the session `dev_start` started. The MCP equivalent of `vibeview dev-stop`. |

**`dev_start` is stateful — it changes the default session for every later
call.** Once it succeeds, any tool called without an explicit `session_id`
targets that session instead of whatever `vibeview dev --detach` state exists
in the project directory. Only one such session can be held at a time
(`dev_start` fails if one is already running), and it stays live — billing
streaming minutes — until `dev_stop` or the MCP connection ends. Always call
`dev_stop` when you finish verifying.

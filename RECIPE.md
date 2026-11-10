# Dictate

Local dictation for Herdr on macOS. One key starts recording, the same key stops it; the speech is transcribed on
the Mac and pasted into the focused terminal without Enter. Other mods (the chat bar's mic) can ask for a recording
and get the text back through shared state.

**What you see:** nothing while idle (the float renders an empty `div`, so Herdr hides it). While recording, a
floating block at the bottom center, drawn by Herdr's float frame, holding one 30 px row with 10 px side padding,
8 px gaps, 12.5 px text in `foreground`:
- a 9 px `blocked`-colored dot pulsing (opacity 0.35 → 1, 1 s period);
- "Recording 0:12" (at least 70 px wide), the time since recording started;
- "Stop" and "✕" as 22 px text buttons in `muted`, 6 px corners, `hover` background (tooltips: "Stop and transcribe
  (the dictation key does the same)", "Cancel: nothing is typed").
While transcribing: a braille spinner in `muted` (90 ms frames) and "Transcribing…". Errors and hints are toasts.

**How it behaves:**
- `dictate.toggle` (bound to `cmd-shift-m` in the sample keys): idle → start; recording → stop and transcribe;
  starting or transcribing → ignored. `dictate.cancel` and ✕ stop everything, nothing typed.
- Start runs `python3 mods/dictate/voice.py start ~/.config/herdr-app/dictate.config.json`: ffmpeg records 16 kHz
  mono from avfoundation `:<device>` to a wav in `$TMPDIR/herdr-dictate-<uid>/`, at most `maxSeconds`; a worker
  (parakeet-mlx's own venv python, from the shebang of the `parakeet-mlx` launcher) loads the model and transcribes
  a second of silence to warm MLX up. `HF_HUB_OFFLINE=1` is set when the model is already in the Hugging Face cache.
- Stop runs `voice.py stop`: SIGINT to ffmpeg (so the wav header is written), then the wav path goes into a GO
  file; the worker transcribes it, writes OUT atomically and exits. Under a quarter second of audio: "No recording.
  Does Herdr have microphone access?".
- The text becomes one line (line breaks and control characters removed). Then: a request's `reply` gets
  `{...value, text}` at its shared key; else, with a terminal focused (`focused().terminal`), the text goes on the
  clipboard and `terminal.paste` runs (a bracketed paste: no Enter), and 500 ms later the previous clipboard text is
  restored if the clipboard still holds the dictation; with no terminal focused it stays on the clipboard and a
  toast says "paste it with ⌘V". Empty transcript: "Nothing heard".
- On start it runs `voice.py status` and publishes `dictate.api = {version: 1, ready, hint}`; not ready, the key
  shows the hint as a toast ("Dictation needs: brew install ffmpeg · uv tool install parakeet-mlx", or "macOS only").
- `dictate.state = {status, id, since}` follows every change; `dictate.request` is taken once on the next render
  and reset to `null` (PROVIDERS.md).
- Closing the copy mid-recording runs `voice.py cancel`. The worker exits after one job or `maxSeconds + 120` s.

**Built from:** a mod `mods/dictate` with `mod.json` `{ "place": "none" }`, placed by each layout's `floating` list:
`{ "id": "dictate", "content": { "mod": "dictate" }, "edge": "bottom", "align": "center" }`; `main.js` (the view,
the command loop, the request taker), `route.js` (pure: request parsing, `oneLine`, where the text goes, the clock,
the setup hint), `voice.py` (python3 standard library: status, start, stop, cancel, transcribe a file, and the
worker that imports `parakeet_mlx`). It uses `shared() setShared() exec() appHome() focused() run() notify()
theme() nextCommand()`, `cx.read_from_clipboard()`/`cx.write_to_clipboard()`, `cx.timer` for the recording clock
only, the command `terminal.paste`, and runs `python3`, `ffmpeg` and parakeet-mlx's python.

**Watch out:**
- `hc send` always presses Enter and `hc keys` rejects non-ASCII keys: typing goes through the clipboard and
  `terminal.paste`.
- `<mod>.<command>` only reaches a copy in the shown layout: the float must be in every layout where the key should
  work. An idle float that renders an empty `div` still receives it.
- `exec` waits for its program: `voice.py start` detaches ffmpeg and the worker (`start_new_session`, stdio to
  /dev/null or a log file) and returns at once.
- Never leave anything running: ffmpeg's `-t maxSeconds`, the worker's TTL, cancel on ✕ and on `close()`.
- The model's first load without a cache downloads it; don't force `HF_HUB_OFFLINE` then.
- The recording clock is data (1 s timer while recording, cancelled at stop); the dot and spinner animate natively
  (`.loop`, `.frames`).
- Herdr needs macOS microphone permission; ffmpeg inherits it from the app.

**Done when:** `herdr-app check mods/dictate` passes; `node --test mods/dictate/test_route.mjs` passes; `python3
mods/dictate/voice.py transcribe hello.wav <config>` on a `say`-generated file prints `{"ok": true, "text": "Hello
world…"}` and leaves no `voice.py worker` process; with the float in the layout and the key bound, pressing it shows
"Recording", pressing it again pastes the words at the cursor of the focused terminal without submitting; and
`setShared("dictate.request", {action: "toggle", reply: {key: "x", value: {}}})` twice puts `{text}` at `x`.

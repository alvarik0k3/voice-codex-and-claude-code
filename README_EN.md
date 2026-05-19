# Voice → Claude / Codex with Handy

Voice-powered AI assistant for Linux using **Handy** (Parakeet STT) as the local transcription engine, routed to **Claude** or **Codex CLI** with **piper TTS** — integrated into Hyprland via an external script bridge.

---

## How it works

```
Super+Z (press)
  → handy-mode writes "claude-speak" to /tmp/handy-ai-mode
  → triggers: handy --toggle-transcription   (Handy starts recording)

(you speak)

Super+Z (press again)
  → handy --toggle-transcription   (Handy stops recording)
  → parakeet transcribes in ~200ms
  → Handy calls handy-ai-bridge with the transcribed text
  → bridge reads the mode → sends to Claude CLI
  → Claude responds → piper TTS reads it out loud
```

`Ctrl+Space` works as plain dictation (no AI).

---

## Keyboard shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+Space` | Plain dictation — transcribes and types where the cursor is |
| `Super+Z` | Ask **Claude** → responds with **voice** |
| `Super+Shift+Z` | Ask **Claude** → **types** the response into the active window |
| `Super+Q` | Ask **Codex** → responds with **voice** |
| `Super+Shift+Q` | Ask **Codex** → **types** the response into the active window |

---

## System files

These are the files actually in use. The ones in this folder are reference copies only.

| File | Real location | Description |
|------|--------------|-------------|
| `handy-ai-bridge` | `~/.local/bin/handy-ai-bridge` | Main Python script — receives transcription from Handy and routes it |
| `handy-mode` | `~/.local/bin/handy-mode` | Bash helper — writes the mode and triggers Handy |
| Hyprland bindings | `~/.config/hypr/bindings.lua` | Keyboard shortcuts |
| Handy config | `~/.local/share/com.pais.handy/settings_store.json` | `paste_method: external_script` + bridge path |

---

## Installation from scratch

### 1. System dependencies

> The Python script (`handy-ai-bridge`) uses only the standard library — no pip packages needed. All you need are these system tools:

```bash
# Handy — voice transcription app (AUR)
yay -S handy

# wtype — types text into Wayland windows
sudo pacman -S wtype

# aplay — audio playback (usually already installed)
sudo pacman -S alsa-utils

# notify-send — desktop notifications
sudo pacman -S libnotify

# Claude Code CLI — Claude AI assistant
# Follow instructions at https://claude.ai/code

# Codex CLI — OpenAI GPT-4o assistant
# Follow OpenAI Codex CLI instructions

# piper TTS — local text-to-speech
# 1. Download the binary from https://github.com/rhasspy/piper/releases
#    and place it at ~/.local/bin/piper
# 2. Download a voice model:
#    https://github.com/rhasspy/piper/blob/master/VOICES.md
#    Recommended: es_ES-davefx-medium (or any language you prefer)
#    Place the .onnx and .onnx.json files in ~/.local/share/piper/voices/
```

### 2. Copy the scripts

```bash
cp handy-ai-bridge ~/.local/bin/handy-ai-bridge
cp handy-mode ~/.local/bin/handy-mode
chmod +x ~/.local/bin/handy-ai-bridge ~/.local/bin/handy-mode
```

### 3. Configure Handy

Stop Handy if running:
```bash
pkill handy
```

Edit `~/.local/share/com.pais.handy/settings_store.json` and change:
```json
"paste_method": "external_script",
"external_script_path": "/home/YOUR_USER/.local/bin/handy-ai-bridge",
```

Restart Handy:
```bash
handy --start-hidden &
```

### 4. Configure Hyprland

In `~/.config/hypr/bindings.lua`, add or replace the voice assistant lines:

```lua
-- Plain dictation with Ctrl+Space
o.bind("CTRL + SPACE", "Handy: transcribe voice", "handy --toggle-transcription")

-- AI voice assistants via Handy
o.bind("SUPER + Z",         "Claude Assistant (voice)",  "handy-mode claude-speak")
o.bind("SUPER + SHIFT + Z", "Claude Assistant (type)",   "handy-mode claude-type")
o.bind("SUPER + Q",         "Codex Assistant (voice)",   "handy-mode codex-speak")
o.bind("SUPER + SHIFT + Q", "Codex Assistant (type)",    "handy-mode codex-type")
```

Reload Hyprland:
```bash
hyprctl reload
```

---

## Why Handy instead of faster-whisper

| | faster-whisper (previous) | Handy / parakeet (current) |
|--|--|--|
| Speed | ~3-5s (reloads model every call) | ~200ms (model stays in RAM) |
| Spanish support | Yes (multilingual) | Yes (multilingual) |
| Script integration | Yes (stdout) | Via `external_script_path` |
| Model | Whisper small 462MB | parakeet-tdt-0.6b-v3 ONNX 640MB |

---

## How handy-ai-bridge works internally

1. Handy calls the script with the transcribed text (via stdin or argv)
2. The script checks for `/tmp/handy-ai-mode`
   - **Does not exist** → plain dictation, uses `wtype` to type the text
   - **Exists** → reads the mode (`claude-speak`, `claude-type`, `codex-speak`, `codex-type`) and deletes it
3. Sends the text to Claude or Codex CLI
4. Based on the mode:
   - `speak` → synthesizes the response with piper TTS and plays it with aplay
   - `type` → types the response into the active window with wtype

---

## Changing the Handy model

Open the Handy app, go to Settings → Model, download the desired model and set it as default. No script changes needed — the bridge receives the already-transcribed text regardless of which model Handy uses.

---

## Troubleshooting

**Shortcut does nothing:**
```bash
pgrep handy           # check Handy is running
handy --start-hidden &  # start it if not running
```

**Transcription error:**
```bash
tail -20 ~/.local/share/com.pais.handy/logs/handy.log
```

**Bridge not calling Claude/Codex:**
```bash
# Test the bridge manually
echo "what is Python" | ~/.local/bin/handy-ai-bridge
# Check that the notification and response appear
```

**Stale mode file in /tmp:**
```bash
rm -f /tmp/handy-ai-mode
```

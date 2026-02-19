# codex-speach2text-script

Small shell scripts that glue together **PipeWire**, **whisper.cpp**, **WezTerm**, and **Codex CLI** so you can speak a request and have it land as a clean prompt inside your running **interactive Codex** pane.

## What it does

Typical flow:

1. Record audio from a PipeWire source (`pw-record`)
2. Transcribe locally using `whisper.cpp`
3. Optionally “compress” the transcript into a crisp instruction via `codex exec`
4. Auto-detect the WezTerm pane running interactive Codex (tagged `CODEX_TUI`)
5. Paste the final prompt into that pane and press Enter

## Scripts

- `codex_ptt_start` — starts recording to `${XDG_RUNTIME_DIR:-/tmp}/codex_ptt/voice.wav`
- `codex_ptt_stop` — stops recording, transcribes, compresses, auto-detects the `CODEX_TUI` pane, and sends the final prompt
- `codex_ptt_toggle` — convenience toggle (start/stop) for binding to a single hotkey
- `codex` — wrapper used to tag the interactive Codex pane title as `CODEX_TUI` (so auto-detection works)
- `voice2prompt` / `voice2codexprompt` — earlier experiments / alternatives for the pipeline

## Requirements

Core:

- Linux + **PipeWire** (`pw-record`)
- **WezTerm** (`wezterm cli`)
- **jq**
- **whisper.cpp** built locally + a model
- **Codex CLI** (`codex`)

Optional:

- `notify-send` (desktop notifications; usually from `libnotify`)

> Your window manager / desktop environment is **not** a dependency. You only need *some* way to bind a key to run `codex_ptt_toggle`.

## Install

Clone and put the scripts on your PATH:

```bash
git clone https://github.com/benjiinzz/codex-speach2text-script
cd codex-speach2text-script

mkdir -p ~/.local/bin
cp codex codex_ptt_* voice2* ~/.local/bin/
chmod +x ~/.local/bin/codex ~/.local/bin/codex_ptt_* ~/.local/bin/voice2*
```

Make sure `~/.local/bin` is in your PATH (bash):

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Install `jq` if needed:

```bash
# Debian/Ubuntu
sudo apt install -y jq

# Arch
sudo pacman -S --needed jq

# Fedora
sudo dnf install -y jq
```

## whisper.cpp setup

Build whisper.cpp and download a model:

```bash
mkdir -p ~/src && cd ~/src
git clone https://github.com/ggml-org/whisper.cpp
cd whisper.cpp
cmake -B build
cmake --build build -j
./models/download-ggml-model.sh base.en
```

The scripts default to:

- `~/src/whisper.cpp/build/bin/whisper-cli`
- `~/src/whisper.cpp/models/ggml-base.en.bin`

Override the base directory if yours differs:

```bash
export WHISPER_DIR="/path/to/whisper.cpp"
```

## Pick your microphone source (PipeWire)

Find your source id:

```bash
wpctl status
```

Then edit `~/.local/bin/codex_ptt_start` and set:

```bash
MIC_ID="59"
```

(Replace `59` with your actual source id.)

## Make the Codex pane detectable (CODEX_TUI)

WezTerm doesn’t expose “running command” in `wezterm cli list`, so this repo uses a simple trick: **tag the interactive Codex pane** by setting its title to `CODEX_TUI`.

Use the included wrapper:

1. Start a **fresh shell** (so the wrapper is picked up from `~/.local/bin`).
2. `cd` into a repo you trust and run:

```bash
codex
```

Verify WezTerm sees it:

```bash
wezterm cli list | grep CODEX_TUI
```

> If your prompt theme constantly overwrites terminal titles, keep using the wrapper and disable title-overwriting prompt features.

## Bind a hotkey (WM/DE agnostic)

Bind a key to run:

```bash
~/.local/bin/codex_ptt_toggle
```


Behavior:

- First press: start recording
- Second press: stop → transcribe → compress → send into the `CODEX_TUI` pane

### Example: Hyprland

Add to `~/.config/hypr/hyprland.conf`:

```conf
bind = SUPER, V, exec, /home/<you>/.local/bin/codex_ptt_toggle
```

Reload:

```bash
hyprctl reload
```

### Example: Sway

In `~/.config/sway/config`:

```conf
bindsym $mod+v exec ~/.local/bin/codex_ptt_toggle
```

Reload sway config.

### Example: KDE / GNOME

Use your system’s “Custom Shortcuts” UI to run `~/.local/bin/codex_ptt_toggle`.

### Alternative: Hold-To-Talk

Use either native Sway:
```
bindsym {keybind} exec ~/.local/bin/codex_ptt_start
bindsym --release {keybind} exec ~/.local/bin/codex_ptt_stop
```
Or keyd:

```
[main]
{keybind} = overload(codexptt, rightalt)

[codexptt]
{keybind} = macro(exec ~/.local/bin/codex_ptt_start)
{keybind}:up = macro(exec ~/.local/bin/codex_ptt_stop)
```

Use `~/.local/bin/codex_ptt_start`

## Usage

1. In WezTerm, open a pane in your project directory and run `codex` (interactive).
2. Hit your hotkey once → recording starts.
3. Speak your request.
4. Hit hotkey again → recording stops, transcription/compression runs, and the final prompt is pasted into Codex.

## Logs & artifacts

Each recording cycle overwrites a single log file:

- `${XDG_RUNTIME_DIR:-/tmp}/codex_ptt/ptt.log`

Audio + transcript:

- `${XDG_RUNTIME_DIR:-/tmp}/codex_ptt/voice.wav`
- `${XDG_RUNTIME_DIR:-/tmp}/codex_ptt/voice.txt`

Debug:

```bash
tail -n 200 "${XDG_RUNTIME_DIR:-/tmp}/codex_ptt/ptt.log"
```

## Notes

- “Untrusted folder” warnings: `codex_ptt_stop` runs `codex exec` from the detected Codex pane’s CWD to avoid this.
- The “compress prompt” step is intentionally simple (single-instruction output). Tweak the `COMPRESS_PROMPT` string in `codex_ptt_stop` if you want different formatting.



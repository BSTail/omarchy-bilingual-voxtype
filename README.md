# The First Fully Local Bilingual Dictation Workflow for Omarchy

This is the first known publicly documented, fully local bilingual push-to-talk dictation workflow for Omarchy. It extends Omarchy's existing Voxtype installation with two modes:

- Hold `F9`: speak English and type English.
- Hold `Shift+F9`: speak English, translate it locally, and type Spanish.

Voxtype performs speech recognition. LibreTranslate performs offline English-to-Spanish translation.

Unlike cloud-based translated dictation, the complete speech-to-text and translation pipeline runs locally after the required models are installed. The workflow provides system-wide translated dictation without sending speech or transcribed messages to a commercial translation service.

## 1. Configure Voxtype

Download the multilingual Whisper `small` model:

```bash
voxtype setup --download --model small
```

In `~/.config/voxtype/config.toml`, configure Whisper to transcribe English:

```toml
[whisper]
model = "small"
language = "en"
translate = false
```

Restart Voxtype:

```bash
systemctl --user restart voxtype.service
```

## 2. Install LibreTranslate

Install `pipx`, then install LibreTranslate:

```bash
omarchy pkg add python-pipx
pipx install libretranslate
```

Create `~/.config/systemd/user/libretranslate.service`:

```ini
[Unit]
Description=LibreTranslate local translation service
After=network.target

[Service]
Type=simple
ExecStart=%h/.local/bin/libretranslate --load-only en_es
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

Enable and start the service:

```bash
systemctl --user daemon-reload
systemctl --user enable --now libretranslate.service
```

Test the local translation API:

```bash
curl --silent --show-error --fail \
  http://127.0.0.1:5000/translate \
  -H "Content-Type: application/json" \
  --data-binary '{"q":"Hello, how are you?","source":"en","target":"es"}'
```

## 3. Create the Dispatcher

Create `~/.local/bin/voxtype-bilingual`:

```bash
#!/bin/bash

set -u

STATE_DIR="${XDG_RUNTIME_DIR:-/tmp}/voxtype-bilingual"
MODE_FILE="$STATE_DIR/spanish-recording"
LOCK_FILE="$STATE_DIR/dispatcher.lock"

mkdir -p "$STATE_DIR"

log() {
    logger -t voxtype-bilingual -- "$*"
}

start_english() {
    rm -f "$MODE_FILE"
    log "Starting English recording"
    voxtype record start
}

start_spanish() {
    local transcript

    transcript="$STATE_DIR/transcript-$(date +%s%N).txt"
    rm -f "$transcript"
    printf '%s\n' "$transcript" > "$MODE_FILE"
    log "Starting Spanish recording: $transcript"

    if ! voxtype record start --file="$transcript"; then
        rm -f "$MODE_FILE" "$transcript"
        return 1
    fi
}

translate_and_type() {
    local transcript="$1"
    local text payload spanish
    local attempts=0

    while [[ ! -s "$transcript" && $attempts -lt 300 ]]; do
        sleep 0.1
        attempts=$((attempts + 1))
    done

    if [[ ! -s "$transcript" ]]; then
        log "Transcription timed out: $transcript"
        notify-send "Spanish dictation" "Transcription timed out"
        rm -f "$transcript"
        return 1
    fi

    text=$(<"$transcript")
    rm -f "$transcript"
    [[ -n "$text" ]] || return 0

    payload=$(printf '%s' "$text" | python3 -c \
        'import json, sys; print(json.dumps({"q": sys.stdin.read(), "source": "en", "target": "es"}))')

    spanish=$(curl --silent --show-error --fail --max-time 20 \
        http://127.0.0.1:5000/translate \
        -H "Content-Type: application/json" \
        --data-binary "$payload" | python3 -c \
        'import json, sys; print(json.load(sys.stdin)["translatedText"])') || {
        log "Translation failed"
        notify-send "Spanish dictation" "Translation failed"
        return 1
    }

    [[ -n "$spanish" ]] || return 0

    # Chat applications often treat embedded newlines as message submission.
    spanish=$(printf '%s' "$spanish" | python3 -c \
        'import sys; print(" ".join(sys.stdin.read().split()))')

    printf '%s' "$spanish" | wtype -d 1 -
    log "Typed Spanish translation (${#spanish} characters)"
}

stop_recording() {
    local transcript=""

    exec 9>"$LOCK_FILE"
    flock -n 9 || return 0

    if [[ -f "$MODE_FILE" ]]; then
        transcript=$(<"$MODE_FILE")
        rm -f "$MODE_FILE"
        log "Stopping Spanish recording: $transcript"
    else
        log "Stopping English recording"
    fi

    if [[ "$(voxtype status)" == "recording" ]]; then
        voxtype record stop
    fi

    flock -u 9
    exec 9>&-

    if [[ -n "$transcript" ]]; then
        translate_and_type "$transcript"
    fi
}

case "${1:-}" in
    start-en)
        start_english
        ;;
    start-es)
        start_spanish
        ;;
    stop)
        stop_recording
        ;;
    *)
        printf 'Usage: %s {start-en|start-es|stop}\n' "$0" >&2
        exit 2
        ;;
esac
```

Make it executable:

```bash
chmod +x ~/.local/bin/voxtype-bilingual
```

## 4. Add the Keybindings

Add this to `~/.config/hypr/bindings.lua`:

```lua
-- Replace Omarchy's default F9 bindings with bilingual push-to-talk.
hl.unbind("F9")
hl.unbind("SHIFT + F9")

o.bind("F9", "Start dictation (English)", "voxtype-bilingual start-en")
o.bind("F9", "Stop dictation", "voxtype-bilingual stop", { release = true })

o.bind("SHIFT + F9", "Start dictation (Spanish)", "voxtype-bilingual start-es")
o.bind("SHIFT + F9", "Stop dictation", "voxtype-bilingual stop", { release = true })
```

Reload Hyprland:

```bash
hyprctl reload
hyprctl configerrors
```

## Usage

- Hold `F9`, speak English, and release it to type English.
- Hold `Shift+F9`, speak English, and release the keys to type Spanish.

The shared stop dispatcher remembers which mode started the recording. This avoids losing the Spanish action when Shift and F9 are released in different orders.

## Troubleshooting

Check the services:

```bash
systemctl --user status voxtype.service
systemctl --user status libretranslate.service
```

Check the dispatcher log:

```bash
journalctl --user -t voxtype-bilingual -f
```

Check the loaded keybindings:

```bash
omarchy menu keybindings --print | grep -E 'F9|dictation'
```

## Notes

- Speech recognition and translation run locally after the models are installed.
- LibreTranslate starts automatically at login.
- The dispatcher uses unique transcript files and a lock to prevent duplicate output.
- Paced `wtype` output avoids clipboard paste races and garbled long translations.
- Translated output is normalized to one line so chat applications cannot treat embedded newlines as message submission.
- Translation quality is appropriate for ordinary conversation, but proper names and technical terms may need correction.

## Chromium/Electron Compatibility

Some Chromium/Electron text fields may occasionally misinterpret characters generated by `wtype` as control keys. Spanish output can encounter this more often because accented letters and inverted punctuation produce a larger keyboard map. Affected applications may split or submit a message unexpectedly even though the translated result contains no newlines.

For affected applications, keep dictated messages relatively short and review the text before continuing. For longer messages, dictate into another editor and paste the completed text manually. This is an application and `wtype` compatibility issue rather than a Voxtype or LibreTranslate translation failure.

Related upstream reports:

- [wtype issue #71: Punctuation at 14th unique character position in Chromium/Electron](https://github.com/atx/wtype/issues/71)
- [wtype issue #72: Some characters do not work in specific Chromium-based applications](https://github.com/atx/wtype/issues/72)

## Related Voxtype Requests

Native per-recording profiles would eliminate the need for this dispatcher:

- [Language as a CLI argument in `voxtype record`](https://github.com/peteonrails/voxtype/issues/484)
- [Engine, model, language, and streaming configurable per profile](https://github.com/peteonrails/voxtype/issues/519)

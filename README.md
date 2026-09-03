> [!NOTE]
> **2026-09-03** — back in service. Archived 2026-08-20 in favour of [niri-noctalia](https://github.com/aier9500/niri-noctalia); the Niri-side improvements from that repo have been mirrored here.

> [!CAUTION]
> The commands in this README assume the repo lives at `~/.dotfiles/niri-dms`.
> Clone it there with the command below, or adjust the `repo=` line in each block.

> [!TIP]
> This repo uses symlinks to their respectives folders in `~/.config` (e.g. ~/.config/niri, ~/.config/voxtype). You can their configs in one place in `~/.dotfiles/niri-dms`.

# Niri + DMS dotfiles

## Features

- Be able to have a polished Niri experience in ~15 minutes.
- Sane and tested keyboard shortcuts (partially inspired by Windows).
- UI tuning that is:
  - Clean & modern.
  - Out-of-the-way with fast but lively animations.
- Voxtype (dictation) setup, and optional local LLM cleanup layer.
- Edit (almost) everything in this repo, all in one place!

## To clone this repo

```bash
mkdir -p ~/.dotfiles
git clone https://github.com/aier9500/niri-dms ~/.dotfiles/niri-dms
```

## Install Niri & DMS

### Installing packages on Fedora

```bash
# Terra repo (terra-extras) has newer versions
sudo dnf install niri DankMaterialShell
dms setup                                        # writes ~/.config/niri/dms/*.kdl
systemctl --user add-wants niri.service dms      # DMS starts and stops with niri
```

Relogin. (`Mod+F4` quits niri.)

### Backs up your current `~/.config/niri` to `~/.config/niri.bak`

**Run once / run only on non-symlink folders.** Running this on a symlinked folder would cause nesting.

```bash
mv ~/.config/niri ~/.config/niri.bak
```

### Scaffold, then link repo Niri configs to `~/.config/niri`

> [!NOTE]
> The repo supplies you with config templates in `niri/template`. Use the code block below to create your editable copies at `niri/user` (gitignored).
>
> The `dms/*.kdl` files are also gitignored as DMS rewrites them (colours, cursor, outputs, GUI window rules) as you use its GUI. The stub loop below creates empty placeholders so Niri won't error on a missing include before DMS has written the real ones. `dms/binds.kdl` and `dms/layout.kdl` are deliberately not included: binds live in `user/binds.kdl`, geometry in `user/theme.kdl`.

Symlinks repo Niri configs to `~/.config/niri`. Creates local copies of every `niri/template/*.kdl` in `niri/user/`.

```bash
repo=~/.dotfiles/niri-dms/niri

ln -sfn "$repo" ~/.config/niri  # symlink repo niri/ configs to .config/niri
cp -rn "$repo/template/." "$repo/user/"  # copy templates

# Stub loop to create DMS config placeholders
mkdir -p "$repo/dms"
for file in alttab colors cursor outputs windowrules; do
  if [ ! -e "$repo/dms/$file.kdl" ]; then
    touch "$repo/dms/$file.kdl"
  fi
done
```

> [!IMPORTANT]
> On Multi-GPU (e.g. laptop with dGPU) check `user/hardware.kdl` to render Niri on the iGPU and avoid dGPU wake-up lag. Single-GPU machines can leave it as is.

> [!TIP]
> Monitors (refresh rate, layout, scale) are set in DMS Settings → Displays, which writes `dms/outputs.kdl`.

> [!TIP]
> Per-app window rules: `Mod+Shift+W` opens the DMS editor, which writes `dms/windowrules.kdl`.

### Nvidia High VRAM Fix

The Nvidia driver doesn't return freed VRAM to Niri, so Niri can hog ~1 GiB VRAM instead of ~100 MiB. The driver ships a fix profile it is not wired automatically for Niri (Smithay-based compositors), so we have to apply it ourselves.

```bash
sudo mkdir -p /etc/nvidia/nvidia-application-profiles-rc.d/
sudo tee /etc/nvidia/nvidia-application-profiles-rc.d/50-limit-free-buffer-pool-in-wayland-compositors.json > /dev/null << 'EOF'
{
  "rules": [
    {
      "pattern": {
        "feature": "procname",
        "matches": "niri"
      },
      "profile": "Limit Free Buffer Pool On Wayland Compositors"
    }
  ],
  "profiles": [
    {
      "name": "Limit Free Buffer Pool On Wayland Compositors",
      "settings": [
        {
          "key": "GLVidHeapReuseRatio",
          "value": 0
        }
      ]
    }
  ]
}
EOF
sudo chmod 644 /etc/nvidia/nvidia-application-profiles-rc.d/50-limit-free-buffer-pool-in-wayland-compositors.json
```

Restart Niri to apply. See the [Niri Nvidia wiki](https://github.com/niri-wm/niri/wiki/Nvidia).

### fcitx5 + Rime Setup (Keyboard Layout Management, Chinese)

Optional — skip if you don't use multiple keyboard layouts. This repo's Niri configs autostarts fcitx5 and sets `XMODIFIERS` X11 compatibility already (see `niri/user/autostart.kdl` & `misc.kdl`); the actual layout config and and hotkey are managed in `fcitx5` itself (`fcitx5-configtool` GUI available).

```bash
sudo dnf install fcitx5 fcitx5-rime fcitx5-configtool
```

## DMS Settings Suggestions

Some settings worth changing in the DMS settings after fresh install. Sorted by section.

> [!TIP]
> This repo's keyboard shortcuts live in `niri/user/binds.kdl`.
>
> I highly suggest not using the DMS GUI for bindings. Consider binding via the .kdl file and backing them up.

### Personalization

- Theme & Color
  - Theme Color -> Auto (derived from wallpaper)
  - Automatic Color Mode
    - Automatic Control
    - Share Gamma Control Settings
  - Font
- App theming -> enable GTK, Qt (qt6ct), Ghostty, Kitty

### Dank Bar

- Position -> bottom, floating with 8px margins, ~70% opacity
- Widgets:
  - Left
    - Workspace Switcher (no labels)
    - Running Apps (current workspace only)
  - Centre
    - Media Controls (hide when idle)
  - Right
    - System Tray
    - dGPU Sleep Monitor _plugin_
    - Battery
    - Control Center
    - Clock (`%Y-%m-%d · %H:%M` aka ISO)
    - Notification Center

- `arqueon-dms-gaze-auth` (face unlock; alternative to the PAM edit below)

### Displays

- Gamma Control
  - Night 3500k (suggested ambient lighting 2700k-3000k)
  - Day 6500k (suggested ambient lighting 4000k-5000k)
  - Schedule 05:30 -> 18:00
- Brightness -> enable ddcutil for external monitors

### Power & Security -> Power & Sleep -> Idle Settings

- Lock before suspend -> true
- Lock 5m, screen off 10m, sleep 20m

## Voice Typing (`voxtype`)

Press hotkey, talk, and the text pastes at your cursor. An optional local-LLM tidies the grammar and spelling.

> [!TIP]
> `voxtype` is the solution I would suggest before OpenWhispr implements better Wayland compositor support.
>
> OpenWhispr has a much simpler setup but its universal keybinding does not work without an extensive amount of tweaks.

### Install `voxtype`

Install the package using instructions from [https://voxtype.io/download](https://voxtype.io/download). Then:

Create the `voxtype` config:

```bash
voxtype setup
```

> [!TIP]
> **Whispr model recommendations:**
>
> GPU: `large-v3-turbo`\
> Capable iGPU: `small`\
> Smaller models: `base`, `tiny`

Configure a transcription model:

```bash
voxtype setup model # then select your model
```

```bash
voxtype setup --download  # download the speech model
voxtype setup check       # report anything still missing
```

### `voxtype` Binding

In this repo, it is bound via `niri/user/binds.kdl`:

- `Alt+Space` -> start / stop recording
- `Alt+Shift+Space` -> cancel, discards without pasting

In the `~/.config/voxtype/config.toml`, disable kernel level shortcut. Disable on Wayland compositors; enable only on DEs like GNOME and KDE.

```toml
[hotkey]
enabled = false
```

### Autostart

Instead of `voxtype setup systemd` which uses the stock daemon, use the repo's unit + launcher so the dictionary comes from `~/.config/voxtype/dictionary.txt`. More on the dictionary in the section below.

The following code

- Creates a private copy of the example dictionary list.
- Links the following to `~/.config/voxtype/`:
  - `voxtype/dictionary.txt` — portable dictionary list.
  - `voxtype/voxtype-with-dictionary.sh` — script that pipes dictionary list into whispr.
- And links the following to `~/.config/systemd/user/`:
  - `voxtype/voxtype.service` — `systemd` service that tells `voxtype` to always use the script above.
- Reloads and starts custom `voxtype`.

```bash
repo=~/.dotfiles/niri-dms/voxtype
mkdir -p ~/.config/systemd/user  # may not exist on fresh install
cp -n "$repo/dictionary.txt.example" "$repo/dictionary.txt"
ln -sfn "$repo/dictionary.txt" ~/.config/voxtype/dictionary.txt
ln -sfn "$repo/voxtype-with-dictionary.sh" ~/.config/voxtype/voxtype-with-dictionary.sh
ln -sfn "$repo/voxtype.service" ~/.config/systemd/user/voxtype.service
systemctl --user daemon-reload
systemctl --user enable --now voxtype    # autostart on login + start now
```

### Configuring `voxtype`

The `voxtype` config file is in `~/.config/voxtype/config.toml`. This is an autogenerated file by `voxtype` so it is not symlinked from this repo.

Edit using:

```bash
$EDITOR ~/.config/voxtype/config.toml
```

... or edit via the TUI using:

```bash
voxtype configure
```

#### Sound feedback

In `~/.config/voxtype/config.toml`, enable sound feedback and set it to a non-distracting volume.

```toml
[audio.feedback]
enabled = true
volume = 0.2
```

#### Output mode

Set the output mode to `clipboard` so the transcript is copied to your clipboard and you paste it yourself with `Ctrl+V` or `Ctrl+Shift+V`. Requires `wl-copy` (Wayland).

The four modes:

- `type` — simulate keystrokes char-by-char at the cursor.
- `clipboard` — copy to clipboard, paste manually.
- `paste` — copy to clipboard, then fires `Ctrl+V`, might not work in the terminal.
- `file` — write the transcript to a file (`file_path` needs to be defined).

```toml
[output]
mode = "clipboard"
```

#### Personal dictionary

Proper nouns and jargon that Whisper would otherwise mistranscribe live in `voxtype/dictionary.txt` — one term per line, `#` comments and blank lines ignored:

```
# voxtype/dictionary.txt
# --- AI / LLM ---
OpenWhispr
Claude
Qwen
# --- People (add your own) ---
Jane Doe
Bob
# ... etc
```

The repo ships an template dictionary list `voxtype/dictionary.txt.example`; the [Autostart](#autostart) step above created a local copy to `voxtype/dictionary.txt`.

Voxtype can't read a word list from a file on its own — it only accepts an `initial_prompt` string in `config.toml`. To avoid cluttering the config file, we use `voxtype/voxtype-with-dictionary.sh` to process and inject `dictionary.txt` to run `voxtype --initial-prompt "…" daemon`.

### QoL Additions to `.bashrc`

Add these to your `~/.bashrc` (or your shell's config — `~/.zshrc`, `~/.config/fish/config.fish`, etc.):

- `edit-dict` to edit the dictionary.
- `restart-voxtype` to restart voxtype after dictionary edit.
- `restart-voxllm` to restart voxtype and ollama to use dGPU after turning GPU on.

```bash
# voxtype QoL aliases
alias edit-dict='$EDITOR $HOME/.config/voxtype/dictionary.txt'
alias restart-voxtype='systemctl --user restart voxtype.service'
alias restart-voxllm='systemctl --user restart voxtype.service && systemctl restart ollama'
```

### AI cleanup (optional)

Runs each transcript through a local LLM to fix grammar, converts to British spelling, strips the "um"s.

> [!TIP]
> This repo has a script (`voxtype/cleanup.sh`) that pipes raw output to Gemma 4 E4B and processes it; it contains the prompt that is given to the LLM to clean up text.
>
> Feel free to tweak the script if 1) you don't want conversion to British English, 2) want specific writing style/tone, 3) others.

> [!WARNING]
> Needs Ollama **0.20+**. Ollama doesn't package for repos, that's why we are curling.
>
> `cleanup.sh` req: `jq` and `curl`.

Installs Ollama, downloads gemma4:e4b model, applies LLM cleanup script:

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull gemma4:e4b
ln -sfn ~/.dotfiles/niri-dms/voxtype/cleanup.sh ~/.config/voxtype/cleanup.sh
```

In `~/.config/voxtype/config.toml`, set the post processing command to use the LLM cleanup script:

```toml
[output.post_process]
command = "sh -c 'exec \"$HOME/.config/voxtype/cleanup.sh\"'"
```

## `gaze` (facial recognition) integration with DMS Lock

`gaze` has yet to add official support for DMS Lock. DMS Lock reads `/etc/pam.d/dankshell`:

```bash
sudo tee /etc/pam.d/dankshell > /dev/null << 'EOF'
#%PAM-1.0
# Let gaze unlock DMS Lock
auth    sufficient    pam_gaze.so
# Fall back to password
auth    substack      system-auth
EOF
sudo chmod 644 /etc/pam.d/dankshell
```

Press enter in the DMS Lock to trigger `gaze`.

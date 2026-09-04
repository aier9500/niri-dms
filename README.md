> [!NOTE]
> **2026-09-03** — back in service. Archived 2026-08-20 in favour of [niri-noctalia](https://github.com/aier9500/niri-noctalia); the Niri-side improvements from that repo have been mirrored here.

> [!CAUTION]
> The commands in this README assume the repo lives at `~/.dotfiles/niri-dms`.
> Clone it there with the command below, or adjust the `repo=` line in each block.

> [!TIP]
> This repo uses symlinks to their respectives folders in `~/.config` (e.g. ~/.config/niri). You can their configs in one place in `~/.dotfiles/niri-dms`.

# Niri + DMS dotfiles

## Features

- Be able to have a polished Niri experience in ~15 minutes.
- Sane and tested keyboard shortcuts (partially inspired by Windows).
- UI tuning that is:
  - Clean & modern.
  - Out-of-the-way with fast but lively animations.
- OpenWhispr (dictation) setup with AI clean-up, fully-local available.
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

### Plugins

**dGPU Sleep Monitor** — Control GPU mode with `cardwire`.
**Gaze Authentification** — Face unlock integration; simpler alternative to the PAM edit below.

### Displays

- Gamma Control
  - Night 3500k (suggested ambient lighting 2700k-3000k)
  - Day 6500k (suggested ambient lighting 4000k-5000k)
  - Schedule 05:30 -> 18:00
- Brightness -> enable ddcutil for external monitors

### Power & Security -> Power & Sleep -> Idle Settings

- Lock before suspend -> true
- Lock 5m, screen off 10m, sleep 20m

## Voice Typing (`OpenWhispr`)

Dictation with text cleanup.

OpenWhispr has no Niri support: it does have a Hyprland backend that exposes a Dbus service and for the compositor to call.

### Install OpenWhispr

Download the package from [https://openwhispr.com](https://openwhispr.com) and install it.

Then install `wtype`, which OpenWhispr uses to paste on wlroots-style compositors:

```bash
sudo dnf install wtype jq
```

Set up OpenWhispr within the GUI.

> [!TIP]
> **Dictation model recommendations:**
>
> GPU: Turbo\
> Capable iGPU: Small\
> Smaller models: Base, Tiny
>
> **Cleanup model recommendations:**
>
> GPU (4-6 GB VRAM): Gemma 4 E4B\
> GPU (6+ GB VRAM): Qwen 3.5 9B\
> Off-device: OpenWhispr Cloud, self-host, no cleanup.
>
> **My setup:**
> 4060 Mobile 8 GB VRAM, Turbo + Gemma 4 E4B.

### Launcher and desktop override

Links the launcher into `~/.local/bin` and overrides the packaged desktop entry so **every** way of starting OpenWhispr goes through it.

> [!IMPORTANT]
> The desktop override is not optional, it's needed to access the DBus service.

```bash
repo=~/.dotfiles/niri-dms/openwhispr

mkdir -p ~/.local/bin ~/.local/share/applications
ln -sfn "$repo/openwhispr-niri" ~/.local/bin/openwhispr-niri

# rewrites the .desktop from /user/share with the patched DBus service to ~/.local/share
sed 's|^Exec=.*|Exec='"$HOME"'/.local/bin/openwhispr-niri %U|' \
  /usr/share/applications/open-whispr.desktop \
  > ~/.local/share/applications/open-whispr.desktop
update-desktop-database ~/.local/share/applications
```

Quit OpenWhispr completely and start it again from your app launcher, so it comes up through the wrapper.

Try launching OpenWhispr via `Alt+Space` focused on another app.

### OpenWhispr Binding

In this repo, it is bound via `niri/user/binds.kdl`:

- `Alt+Space` -> toggle dictation

The bind calls the D-Bus method directly, so it works regardless of what hotkey is configured inside OpenWhispr:

```bash
dbus-send --session --type=method_call --dest=com.openwhispr.App \
  /com/openwhispr/App com.openwhispr.App.Toggle
```

### Autostart

`niri/user/autostart.kdl` starts OpenWhispr with the session:

```kdl
spawn-at-startup "sh" "-c" "exec $HOME/.local/bin/openwhispr-niri"
```

### Window rule

`niri/user/theme.kdl` keeps only the pill and places it at the top-centre of the screen instead of leaving an ugly window; it is also set to not steal your focus.

### QoL Additions to your shell config

Add these to `~/.bashrc`, `~/.zshrc` or `~/.config/fish/config.fish`:

### Side effect

OpenWhispr writes `~/.config/hypr/openwhispr-binds.conf` at startup. Niri never reads it, so it is harmless bloat.

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

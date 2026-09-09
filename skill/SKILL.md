---
name: vm
description: Test things in a disposable Omarchy VM (QEMU/KVM on this machine) through the omavm CLI, started from the clean "fresh" snapshot. Use for "/vm", "test this in the VM", "try it in the vm", "open the VM", or whenever work needs a bare Omarchy or is too risky for the real desktop.
---

# Omarchy VM

A disposable VM with a clean Omarchy install, for testing plugins and system changes without touching the real desktop. Everything in it may break: the `fresh` snapshot stays clean and a new VM is one command. Setup and background are in the repository README next to this skill.

Everything goes through **`omavm`**. Start with `omavm --help`; that is the complete reference. Use the script rather than hand-rolling ssh lines or qemu commands: omavm already knows the pitfalls (systemd unit, gum confirm without a tty, session env in the guest).

## The machine

| | |
|---|---|
| CLI | `omavm` (install, provision, dotfiles, boot, resume, stop, status, save, list, ssh, user, agent, push, pull, shot, screendump, sendkey, hypr, qs, restart-shell, plugin) |
| Snapshot | `fresh`: Omarchy with no disk encryption, autologin into Hyprland, sshd on, passwordless sudo, screensaver off and screen kept awake, dotfiles stowed |
| SSH | `root@localhost` on port 2222 with key auth (omavm handles this) |
| Active disk | `/var/tmp/omarchy-iso-boot.qcow2` (gone after a host reboot; snapshots survive) |
| Specs | KVM, 8 cores, 8 GB RAM, 40 GB disk, Hyprland with Quattro (lua config) |

The guest password is whatever `omavm install` was given (`OMAVM_PASSWORD`, default `omarchy`), for both the user and root. You should never need it through omavm; it is for the rare prompt on screen.

## Standard workflow

```bash
omavm status                 # is something already running?
omavm boot                   # fresh VM from "fresh"; waits for SSH itself
omavm ssh 'uname -a'         # command as root in the guest
omavm shot screen.png        # screenshot, then look at it yourself
omavm stop
```

`omavm boot` refuses when a VM is already running. Use that one instead, and do not reboot unasked, because that discards the active disk. The QEMU window may sit on a workspace the user is not on. Leave it there: you do not need to be on it to look.

## Looking and typing without touching the desktop

If SSH does not come up, do not guess and do not go to the QEMU window. Ask QEMU itself for a picture of the guest screen:

```bash
omavm screendump screen.png
omavm sendkey ret
```

Both go through QEMU's QMP socket, so they work with no SSH, no desktop in the guest, and regardless of which workspace the window is on. That is how you tell a waiting prompt from a slow boot from a hang.

`sendkey` types nothing on the host: it goes straight into the virtual machine and cannot disturb the user's work. Key names are QEMU's (`ret`, `esc`, `spc`, `up`, `a`-`z`), dashes make a chord: `ctrl-alt-f2`.

`screendump` needs a framebuffer, which is absent under GPU acceleration. `omavm install` turns acceleration off by itself; elsewhere use `OMARCHY_VM_GL=0 omavm boot`. When the VM runs accelerated omavm says so and points at `omavm shot`, which is sharper anyway but needs a running session in the guest.

## Secrets: nothing in the VM, everything through the agent

The VM deliberately has no disk encryption, boots through to Hyprland without a password, and has passwordless sudo. Everything on that disk is readable by anyone who gets at it. So never put a key, token or password in it, not even temporarily: a qcow2 keeps deleted files around and the snapshot travels with them.

When something in the guest needs one of the user's keys (pushing to GitHub, say), use `omavm agent`. It connects as the guest user with the host's SSH agent forwarded: the guest can have it sign, never read the key, and that access is gone the moment the command returns.

```bash
omavm agent git -C ~/dotfiles push
omavm agent                          # interactive shell with the agent
```

The agent has to be unlocked on the host, otherwise omavm reports that it offers no keys. The same principle covers tokens (`gh`, APIs): fetch them on the host and pass them as an environment variable on one command, rather than writing them to a file in the guest.

## Rebuilding the snapshot

`omavm install` does a full unattended install from the ISO. It builds a cidata drive (the cloud-init `NoCloud` convention: the Omarchy installer takes its answers from a drive labelled `CIDATA` and skips the wizard) with no `disk_encryption` block, waits for the guest to reboot into the installed system on its own, and then runs `omavm provision`: root key, passwordless sudo, idle toggles, stow and the dotfiles. Needs `mtools` on the host for `mcopy`.

```bash
omavm install
omavm stop && omavm save fresh --force   # --force because "fresh" exists
```

It takes half an hour and is rarely needed. `omavm provision` and `omavm dotfiles` can be run on their own against a running VM. The dotfiles go in as a `git clone` of the working copy, so tracked files only; gitignored private keys stay on the host. The stow packages holding ssh config and background services are left out on purpose: the first would override the forwarded agent, the second starts timers that would back up and sync from a test VM.

## Testing plugins

```bash
omavm boot
omavm plugin ~/path/to/plugin
omavm shot screen.png
```

`omavm plugin` copies the directory, registers the bar widget from `manifest.json` in the guest's `shell.json` and restarts the shell. Always check the result yourself with a screenshot; opening the panel without a mouse goes through the plugin's IPC:

```bash
omavm qs ipc call <plugin-id> open     # id is in manifest.json
```

## GUI and Hyprland in the guest

`omavm hypr` and `omavm qs` run with the right session env. Quattro, so `hyprctl dispatch` expects lua syntax:

```bash
omavm hypr dispatch 'hl.dsp.focus({ workspace = 2 })'
omavm user 'wl-paste'      # any command as the guest user, with session env
```

Synthetic input *inside the guest session* (ydotool, wtype) does not work, so testable behaviour goes through the plugin's IPC methods. `omavm sendkey` is a different thing and does work: it enters the virtual machine at the QEMU level rather than through the guest's compositor. Good for keys that land at a tty or a prompt, not a replacement for a real IPC test.

## Snapshots

`omavm save <name>` refuses while the VM is running: stop it first, then save, then boot again if needed. Overwriting an existing snapshot needs `--force`. Never overwrite `fresh` unasked.

## What you may decide yourself

Inside the guest: everything. Install packages, wreck configs, restart services, reboot, delete files; that is what the VM is for. On the host you decide about booting, stopping and the active disk. Ask before: overwriting or deleting the `fresh` snapshot, deleting anything in the VM directory, and anything that reaches past the VM to the host.

## Pitfalls

- The QEMU window can jump to fullscreen. Ctrl-Alt-F toggles QEMU's own fullscreen, Ctrl-Alt-G releases a grabbed mouse.
- Clipboard sharing between host and guest does not work (sdl display). Files go through `omavm push` and `omavm pull`; if SSH is unreachable, the guest-to-host route is `10.0.2.2` plus a temporary `python3 -m http.server` on the host.
- A host reboot eats the active disk. Work that must survive: `omavm stop` and `omavm save` first, or pull the files out with `omavm pull`.
- Never run QEMU as a loose child process of your session (omavm uses a systemd unit for this); otherwise the window is suddenly gone when that session ends.
- If `omavm shot` hangs without failing, the guest screen is probably blanked: grim then waits forever for a frame. In `fresh` the screensaver is off and the screen stays awake (`omarchy-toggle screensaver-off on` and `omarchy-toggle-idle stay-awake`, both set by `omavm provision`). If you meet a VM where it still happens, run `omavm provision`.
- `NOPASSWD` alone is not enough for a VM that must never ask anything: the installer also writes a plain `ALL` rule, and against that `sudo -v` keeps demanding a password. `omarchy-update` does exactly that and then waits on a prompt on screen. `omavm provision` therefore also sets `Defaults:<user> !authenticate`. If you see that prompt anyway, the VM runs from a snapshot predating that fix: `omavm provision` resolves it.
- When rebuilding the snapshot, check yourself that it really comes up unattended. Without disk encryption the installer sets up no SDDM autologin, so the guest would still stop at a login screen; `omavm provision` handles that autologin.

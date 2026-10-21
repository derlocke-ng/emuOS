# emuOS design

Status: draft, agreed in discussion on 2026-10-21. No code exists yet.

## 1. Goals

- A bootable USB SSD that turns any x86_64 machine into an emulation console.
- Full disk encryption. Unlock without a keyboard where possible, with a keyboard everywhere else.
- Usable with a controller only, except LUKS passphrase entry and debugging.
- Atomic updates with rollback.
- Lightweight: no desktop environment, no Flatpak, native packages only.
- A client of the Kiwi Network: WireGuard, Syncthing, RomM and Nextcloud on Kiwi nodes.
- Save games synced to every device that plays them, backed up, and browsable by hand.
- Two ODROID-GO handhelds (Advance and Super) take part in save sync and ROM downloads.

## 2. Non-goals

- Replacing Batocera for arbitrary hardware. Target is x86_64 with AMD or Intel graphics first. NVIDIA is an optional module later.
- A desktop mode. Debugging happens over SSH or on a TTY.
- Running emuOS itself on the handhelds. They run ROCKNIX.
- Streaming ROMs over a network mount. ROMs are downloaded on demand.

## 3. Decisions

| # | Decision | Why | Rejected |
|---|----------|-----|----------|
| D1 | NixOS, built as a flake in this repo | First-class Clevis with Tang and SSH in the systemd initrd, declarative Syncthing, WireGuard, mounts and kiosk session, generations give rollback, emulators are native packages, `hardware.enableAllHardware` makes the initrd portable | Batocera (Buildroot, no LUKS, no package manager), Bazzite (not wanted), lean Fedora bootc (dracut-sshd unproven, emulators spread over coprs) |
| D2 | ES-DE as frontend, native package, running under cage | Upstream ES-DE, controller-first, launches any app including Kodi, has game-start and game-end hooks and a rescan without restart | Batocera's ES fork (its menus depend on Buildroot scripts), Pegasus, Steam Big Picture |
| D3 | Settings and library app in Godot 4, backend daemon in Python | Godot has built-in gamepad focus navigation and theming, nixpkgs ships editor and export templates for headless CI exports. The daemon keeps privileges and logic out of the UI and is usable over SSH | Qt Quick (Qt 6 has no gamepad module), Python with SDL (hand-rolled UI), Dear ImGui (looks like a dev tool) |
| D4 | Saves sync with Syncthing, hub on a Kiwi node | Plain files, native on ROCKNIX, conflicts kept as side-by-side copies, versioning on the hub is the backup, hub folder is browsable over SFTP and Nextcloud | RomM save sync (conflict model still changing, nested code-based folders, no desktop Linux client), rclone bisync to Nextcloud (conflict-prone) |
| D5 | ROM library on RomM, downloaded on demand from the settings app | REST API with tokens, metadata and box art, exports ES-DE gamelist.xml per platform, Grout client for the handhelds | SFTP or NFS mount (large disc images over VPN are too slow) |
| D6 | LUKS2 with Clevis `sss` threshold 1 over Tang, one TPM2 pin per owned machine, plus the passphrase | Auto unlock at home on cable, auto unlock on own laptops anywhere, passphrase as last resort | TPM only (not portable), Tang only (no unlock away from home) |
| D7 | SSH in the initrd for remote unlock | Feeds the planned `kiwi-server unlock` reverse-tunnel idea, NixOS module is mature | dracut-sshd |
| D8 | Handhelds move from EmuELEC to ROCKNIX | Syncthing, WireGuard and Grout from stock menus, mainline kernel, EmuELEC support for RK3326 is fading | Staying on EmuELEC (rclone scripts only) |
| D9 | Nextcloud shows the save hub via External Storage, read mostly | Nice-to-have browsing without a second sync engine | Nextcloud as the sync engine |

## 4. Architecture

Components on the device:

- `boot`: systemd-boot, unified kernel image, systemd stage 1 initrd with networking, Clevis and sshd.
- `session`: autologin of user `player` on tty1 into cage, cage runs a supervisor script that loops ES-DE.
- `frontend`: ES-DE with a curated theme. Custom systems: `settings` (the Godot app), `kodi`.
- `emulators`: RetroArch with a chosen core set, Dolphin, PCSX2, DuckStation, PPSSPP, melonDS, mGBA, RPCS3 optional. All from nixpkgs, versions pinned by the flake lock.
- `emuosd`: Python daemon, system service, Unix socket, polkit or a dedicated group for privileged actions. Owns network, Bluetooth, VPN, sync status, library downloads, updates, rollback, power.
- `settings app`: Godot 4 export, launched by ES-DE like a game, talks only to `emuosd`.
- `syncthing`: system service as user `player`, folders listed in section 6.
- `wireguard`: `wg-quick` unit, config from the Kiwi enrol bundle, toggled by `emuosd`.
- `hooks`: ES-DE game-start and game-end scripts call `emuosd` to trigger a Syncthing rescan and to warn when a newer remote save has not arrived yet.

Services on Kiwi nodes (see the handoff note): Syncthing hub, Tang, RomM, Nextcloud, SFTP, WireGuard master.

## 5. Disk, encryption and unlock

Layout (disko):

- GPT, ESP 1 GiB, one LUKS2 container, btrfs inside with subvolumes `root`, `nix`, `home`, `var`, `games`.
- `games` holds ROMs, media and saves and gets `nodatacow` off, compression on. Everything is encrypted.

Unlock paths, in the order tried:

1. Tang over wired Ethernet on the Kiwi LAN (`ip=dhcp`, `boot.initrd.network.enable`).
2. TPM2 of a known machine (one pin per machine inside the `sss` policy).
3. SSH into the initrd, operator runs `systemd-tty-ask-password-agent`.
4. Passphrase on the console.

Facts to respect:

- Wi-Fi is not available in the initrd. Tang and SSH unlock are wired only.
- WireGuard is not up in the initrd. Tang must be reachable on the LAN, so it runs on the gateway node.
- The JWE is per device, generated at first boot after the passphrase install, stored outside the Nix store.
- Secure Boot is off on foreign machines. Lanzaboote with own keys is possible on owned machines later.
- Verify in a VM: passphrase prompt still appears in systemd stage 1 when Tang is unreachable. NixOS forbids its own fallback option there and relies on systemd-cryptsetup's behavior.

## 6. Data layout and sync

```
/games
  roms/<es-de system>/...          downloaded from RomM, gamelist.xml and media from RomM export
  saves/
    ra/<system>/<game>.srm         RetroArch in-game saves, sorted by content directory
    ra-states/<core>/...           RetroArch states, sorted by core
    dolphin/ pcsx2/ duckstation/ ppsspp/ melonds/ mgba/   standalone emulator save dirs, configured or symlinked here
  bios/
```

Syncthing folders: `ra-saves`, `ra-states`, one folder per standalone emulator. Folder IDs are fixed in the flake and in the handoff note so every device uses the same ones.

Rules learned from the handheld community:

- Sync in-game saves between different OSes freely. Sync states only between identical cores.
- RetroArch save sorting must be identical on every device. Content-directory sorting for saves, core sorting for states.
- Handhelds use `.stignore` to skip systems they cannot run.
- Quit the game and wait for sync before powering off. `emuosd` delays shutdown until Syncthing reports idle.
- Conflicts appear as `.sync-conflict-*` files. The settings app lists them and offers keep-local or keep-remote.

Hub: Syncthing on a Kiwi node, always on, staggered file versioning, data directory exported to Nextcloud as External Storage and covered by Kiwi backups.

## 7. Settings and library app

Launched from ES-DE as the `settings` system. Fullscreen, same resolution and background color as the frontend theme. Godot theme copies the font, palette and spacing of the chosen ES-DE theme.

Screens, one function each:

- Network: Wi-Fi scan and join, wired status, hostname, IP.
- Bluetooth: pair and forget controllers and headsets.
- VPN: WireGuard on and off, peer status, handshake age.
- Sync: per-folder state, pending items, conflicts, force rescan.
- Library: RomM platforms with covers, download with progress, delete locally, keep-flag, free space per system, "rescan frontend" on exit.
- System: update from the flake, generations list with rollback, logs summary, power and reboot.
- About: versions, device ID for Syncthing, enrol state.

`emuosd` API (Unix socket, JSON lines): `net.*`, `bt.*`, `vpn.*`, `sync.*`, `lib.*`, `sys.*`. Long operations stream progress events. The same calls are exposed as a CLI for SSH use.

## 8. Kiwi integration

emuOS is a client, never a stack host. It consumes:

- WireGuard config from `kiwi-server enroll --group devices`.
- Syncthing hub device ID and folder IDs.
- RomM URL and a client API token.
- Nextcloud app password, optional, for browsing only.
- Tang advertisement, for the first-boot Clevis bind.
- Initrd SSH authorized key of the master, for `kiwi-server unlock`.

Proposed form: one enrol bundle (tar or YAML) dropped into `/var/lib/emuos/enrol/`. `emuosd` reads it on boot and configures the units. Nothing from the bundle goes into the Nix store.

## 9. Install, update, CI

- Install: `disko-install` from any machine with Nix writes directly to the USB SSD and prompts for the passphrase. `nixos-anywhere` over SSH is the alternative for installs driven from kiwi-server. A custom installer ISO built from the same flake, with the closure bundled for offline use, comes once the configuration is stable.
- First boot: create `player`, generate the Clevis JWE from the enrol bundle, bind Syncthing, mark enrolled.
- Update: `system.autoUpgrade` pulling the flake from GitHub inside a reboot window, same idea as the Kiwi fleet. Rollback from the settings app or the boot menu.
- CI: GitHub Actions runs `nix flake check`, builds the system closure and the Godot export, pushes to a binary cache so devices never compile. Renovate or `nix flake update` on a schedule.
- Garbage collection: automatic, keep the last few generations, because the Nix store lives on the stick.

## 10. Handhelds

- Both ODROID-GO units move to ROCKNIX, RK3326 image.
- The Super needs a USB Wi-Fi dongle. Confirm the Advance Black Edition's ESP32-S2 Wi-Fi works on ROCKNIX's mainline kernel before relying on it, otherwise use a dongle there too.
- Copy all saves off the EmuELEC cards first and keep the cards as backup.
- On ROCKNIX: enable Syncthing and join the hub, drop the WireGuard config at `/storage/.config/wireguard/wg0.conf` without a DNS line, install Grout for RomM downloads and keep Grout's save sync off.

## 11. Risks and things to verify

- systemd stage 1 passphrase fallback when Tang is down (section 5).
- ROCKNIX Wi-Fi on the Advance Black Edition.
- Grout on ROCKNIX, listed by RomM's README and homepage but not in the docs.
- nixpkgs ES-DE version lags upstream. Override the source in the flake when needed.
- Nix store size on the SSD. Budget around 6 GB for frontend, emulators, cores and Kodi.
- cage needs a logind session, so autologin on tty1 rather than a plain system service.
- RomM's gamelist export path keys changed in 4.9. Pin the RomM version in the Kiwi module.

## 12. Phases

1. Flake skeleton, VM target, LUKS with passphrase, cage plus ES-DE, RetroArch only.
2. Clevis with Tang and TPM, initrd SSH, portable hardware profile, first USB install.
3. `emuosd` with network, VPN, sync and power. Syncthing folders. Godot app with Network, Sync and System screens.
4. RomM library screen, gamelist and media import, standalone emulators.
5. Handheld migration, Nextcloud view, backups, installer ISO, CI cache.

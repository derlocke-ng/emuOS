# emuOS

A portable, encrypted, controller-only emulation OS for the Kiwi Network.

- Base: NixOS (flake), no desktop environment
- Frontend: ES-DE (native package) in a cage kiosk session, Kodi as a launchable app
- Settings and ROM library app: Godot 4 UI with a Python backend daemon
- Disk: LUKS2, unlocked by Tang (Clevis), per-machine TPM2, or passphrase; SSH unlock in the initrd
- Saves: Syncthing, hub on a Kiwi node, versioned, visible in Nextcloud
- ROMs: RomM library on a Kiwi node, downloaded on demand
- Network: WireGuard client enrolled through kiwi-server

Status: design phase, no code yet. Start with [docs/DESIGN.md](docs/DESIGN.md).
The note for the kiwi-server side is in [docs/kiwi-server-handoff.md](docs/kiwi-server-handoff.md).

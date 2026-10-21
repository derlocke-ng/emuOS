# Note for the kiwi-server session

Context: emuOS (this repo) is a NixOS-based, encrypted, controller-only emulation OS on a USB SSD, plus two ODROID-GO handhelds on ROCKNIX. It is a Kiwi *client*, not a fleet host. It needs the following from the fleet side.

## New modules for kiwi-modules

1. `syncthing`: one container, persistent data volume, ports 22000 TCP and UDP (sync), 21027 UDP (local discovery), GUI 8384 behind the reverse proxy. Staggered file versioning on. Purpose: hub for save games and other gaming files from emuOS and the handhelds. Folder IDs and names should be settable so they match the emuOS flake.
2. `tang`: stateless, port 7500, placed on the gateway node so it is reachable from the LAN at initrd time. WireGuard is not up that early, so the master behind the VPN hop is the wrong place.
3. `romm`: RomM with its database, ROM storage volume, metadata provider keys, `scan.export.gamelist_xml: true`. Pin the version, the export config keys changed in 4.9. emuOS and Grout on the handhelds use client API tokens.

## Nextcloud

Optional: expose the Syncthing data directory to the cloud preset's Nextcloud as External Storage (local), with `filesystem_check_changes` or a periodic `occ files:scan`, so saves are browsable there. Nextcloud is a viewer, not the sync engine.

## Backups

Include the Syncthing hub data directory and RomM's assets in the fleet backup set.

## Enrolment

Plan the client enrol bundle to cover a NixOS client, not only Bluefin. Contents emuOS expects:

- WireGuard client config (group `devices`)
- Syncthing hub device ID and the agreed folder IDs
- RomM URL and client token
- Nextcloud app password (optional)
- Tang advertisement or URL
- The master's initrd SSH public key, for the planned `kiwi-server unlock`

## Unlock

emuOS will offer SSH in its initrd on the wired interface. It is the first real consumer of the roadmap item "Tang/Clevis nested SSS with the reverse-tunnel unlock and `kiwi-server unlock`".

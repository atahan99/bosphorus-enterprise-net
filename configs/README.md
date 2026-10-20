# Device configurations

One file per device, named after its hostname, grouped by site:

| Folder | Devices |
|---|---|
| `lon/` | London edge, core, WAN, access and DMZ switches, voice router, rogue test switch |
| `ist/` | Istanbul router and switch pair |
| `chi/` | Chicago router and switch pair |
| `tor/` | Toronto router and switch pair; warehouse router and switch (`TOR-WH-*`, Phase 11) |
| `internet/` | ISP-A, ISP-B, INET-CORE, INET-SW |
| `servers/` | Server-PT, WLC, phone, IoT server and camera settings (GUI-configured, written as checklists) |

## File types

- **`<HOSTNAME>.cfg`**: the build configuration, split into `! ===== Phase N` sections that match [the build guide](../docs/build-guide.md). Paste a phase at a time in global configuration mode. Lines marked `P0-xx` have their Packet Tracer fallback written next to them.
- **`<HOSTNAME>.running.txt`** (added in Phase 10): the device's final `show running-config`, exported from the `.pkt`, so the repo shows exactly what runs in the lab.

Lab passwords appear as configured; they are lab-only and listed in [lab-credentials.md](../docs/lab-credentials.md).

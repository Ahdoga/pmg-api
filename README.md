# pmg-api (fork)

A personal fork of [`proxmox/pmg-api`](https://git.proxmox.com/?p=pmg-api.git), the Proxmox Mail Gateway API. This is not an official Proxmox release and is not supported by Proxmox Server Solutions GmbH.

## What the fork adds

An optional **backup (failover) destination** on a transport map entry. A relay domain can name a second mail server, and Postfix uses it when the first cannot be reached. Example line in `/etc/pmg/transport`:

```
example.com smtp:[10.0.0.1]:25, [10.0.0.2]:25
```

New optional API properties on `/config/transport`: `backup_host`, `backup_port`, `backup_use_mx`. See `REQUIREMENTS.md` for the full list of behavior.

The change lives on the branch `transport-backup-host`. The web interface (`pmg-gui`) is not changed, so set the backup host with `pmgsh` or the API.

## Control documents

These files keep a coding agent (Claude Code) on the right path. They are fork-only and are never sent upstream.

| File | Owns |
| --- | --- |
| `AGENTS.md` | Working rules, commands, dependency policy. Imports the four specs. |
| `REQUIREMENTS.md` | What the fork must do (`FR-*`, `NFR-*`). |
| `ARCHITECTURE.md` | Components, stack, trust boundaries, dependency rules, toolchain. |
| `SECURITY.md` | Secure-coding rules (`SC-*`). |
| `DESIGN.md` | API, file format, message, and command-line conventions. |
| `REQUIREMENT_TEMPLATE.md` | Template for one atomic requirement or issue. |

If a README summary and `REQUIREMENTS.md` disagree, `REQUIREMENTS.md` wins.

## Building and testing

See the Toolchain table in `ARCHITECTURE.md`. Building needs a Debian trixie machine with the Proxmox repositories. Test every change on a non-production PMG VM first.

## License

GNU Affero General Public License, version 3 or later, as upstream. See `debian/copyright`.

# Requirements: pmg-api fork (backup transport destination)

This file owns WHAT the fork must do. It does not say HOW to build it. It covers only the fork's changes and the rules for staying close to upstream. Everything else in pmg-api keeps its upstream behavior.

Sources: the owner's design discussion of 2026-11-16 (PMG in front of a hosted mail account, with a lab mail server as fallback during a Microsoft 365 outage); the upstream `proxmox/pmg-api` code at commit `e889c13` (package version 9.1.6, Debian trixie); Postfix `transport(5)` as quoted in the Proxmox forum thread "Mail Problem" (forum.proxmox.com/threads/mail-problem.73911). Where the owner told the author to decide, the decision is marked *(delegated decision)*.

## Required Requirement Inputs

- **Project purpose:** A personal fork of the Proxmox Mail Gateway API (`pmg-api`) that adds an optional backup (failover) destination to a transport map entry, so a relay domain can name a second mail server that Postfix uses when the first cannot be reached.
- **Users and actors:** PMG administrators (API role `admin`) and auditors (role `audit`), using the web GUI, `pmgsh`, or the REST API. External actors: Postfix on the PMG node; the primary and backup mail servers; the PMG cluster master and other cluster nodes.
- **Core workflows:** list, read, create, update, and delete a transport map entry; write the entry to `/etc/pmg/transport`; rebuild the Postfix lookup table; sync the file to cluster nodes; deliver mail through the primary, then the backup.
- **Data entities:** transport map entry (domain or email, protocol, host, port, MX-lookup flag, comment, and the new backup host, backup port, backup MX-lookup flag).
- **External integrations:** Postfix (`postmap`, `transport_maps`); the PMG cluster file sync.
- **Authentication and roles:** unchanged from upstream. The API uses PMG tickets and roles `admin` and `audit`. The fork adds no role and no unauthenticated function.
- **Regulatory or privacy constraints:** GNU AGPL version 3 or later. The fork MUST keep the upstream license and copyright notices. The fork is not an official Proxmox product.
- **Target surfaces (web UI, APIs, background jobs):** REST API and `pmgsh` only. The web GUI is a separate upstream repository (`pmg-gui`) and is out of scope for this repository. No background jobs are added.

## Functional Requirements

### 1. Transport map entries

- **FR-1.1** An entry MUST keep working exactly as upstream when it has no backup host: same file line, same API values, same Postfix behavior.
- **FR-1.2** An entry MAY have one backup destination, made of a backup host, an optional backup port, and an optional backup MX-lookup flag.
- **FR-1.3** The backup port MUST default to the primary port when it is not given. *(delegated decision)*
- **FR-1.4** The backup MX-lookup flag MUST default to the primary entry's MX-lookup flag when it is not given. *(delegated decision)*
- **FR-1.5** A backup destination MUST be accepted only for the `smtp` protocol. A request that sets a backup host on an `lmtp` entry MUST fail with a clear error and change nothing.
- **FR-1.6** An update that sets the backup host to an empty string MUST remove the whole backup destination (host, port, and MX-lookup flag).
- **FR-1.7** The system MUST write an entry with a backup destination as one line with two comma-separated destinations, primary first, for example `example.com smtp:[10.0.0.1]:25, [10.0.0.2]:25`.
- **FR-1.8** When reading `/etc/pmg/transport`, the system MUST accept lines with one destination and lines with two. It MUST reject, with a logged warning and no crash, any line with more than two destinations, and any `lmtp` line with more than one.
- **FR-1.9** Reading a file and writing it back MUST give the same entries (round trip), including the backup fields.
- **FR-1.10** The system MUST NOT change the `protocol` value stored in memory while writing the map file.
- **FR-1.11** The system SHOULD treat the backup host like the primary host for network trust: an IP address backup host MUST be added to the PMG trusted-network lists in the same way the primary host is.
- **FR-1.12** When the primary destination cannot be reached, Postfix MUST try the backup destination. Behavior when the primary answers with a temporary (4xx) rejection is UNKNOWN until checked against Postfix documentation for the packaged version.

### 2. API and command line

- **FR-2.1** The list and read operations MUST return the backup fields only when they are set.
- **FR-2.2** The create and update operations MUST accept `backup_host`, `backup_port`, and `backup_use_mx` as optional properties.
- **FR-2.3** `backup_host` MUST be validated with the same address rules as `host`. On update it MAY also be the empty string (FR-1.6).
- **FR-2.4** Existing API clients that send none of the new properties MUST see no change in requests, responses, or errors.
- **FR-2.5** The API definition MUST pass `pmgsh verifyapi`.
- **FR-2.6** New and changed properties MUST have a description in the API schema, so generated documentation and `pmgsh` help show them.

### 3. Cluster behavior

- **FR-3.1** Transport changes MUST be made only on the cluster master, as upstream does.
- **FR-3.2** The existing sync of `/etc/pmg/transport` to other nodes MUST carry the new line format with no change to the sync code.
- **FR-3.3** A node that runs the fork MUST NOT break the cluster when another node still runs upstream code (see Open Questions: an upstream node reading a two-destination line).

### 4. Fork maintenance

- **FR-4.1** Each change MUST be a small, separate commit that could be sent upstream as a patch.
- **FR-4.2** The fork MUST stay mergeable with upstream `master`. It MUST NOT reformat, rename, or reorder code it does not need to change.
- **FR-4.3** The fork MUST NOT remove or edit license or copyright notices, and MUST NOT present itself as an official Proxmox release.
- **FR-4.4** Fork-only files (these control documents) MUST NOT be included in any patch intended for upstream.

## Non-Functional Requirements

### 1. Use Constraints

- **NFR-1.1** The fork is for the owner's own use. Anyone who receives the fork or a package built from it MUST also receive the source under AGPL v3 or later.
- **NFR-1.2** The fork MUST NOT add a feature that bypasses or disables Proxmox subscription checks.

### 2. Compatibility

- **NFR-2.1** The fork MUST work with the Postfix version that `debian/control` requires (`>= 3.7.10` at this commit) and on Debian trixie.
- **NFR-2.2** The fork MUST NOT add a Debian or Perl dependency without a decision recorded in `ARCHITECTURE.md`.
- **NFR-2.3** An existing `/etc/pmg/transport` file MUST load unchanged after upgrading to the fork.

### 3. Security and Privacy

- **NFR-3.1** The fork MUST NOT add an unauthenticated API method.
- **NFR-3.2** Every value written to a Postfix map file MUST be validated first, so it cannot add or change another line.
- **NFR-3.3** The fork MUST NOT log passwords, tickets, or keys.

### 4. Quality and Testability

- **NFR-4.1** The parse and write code for the transport map MUST have an automated test that runs in `src/tests` (see `make -C src/tests check`) and covers FR-1.1 to FR-1.10.
- **NFR-4.2** Perl changes MUST pass `make tidy` (`proxmox-perltidy`) with no further changes.
- **NFR-4.3** A package build (`make deb`) MUST succeed and pass `lintian`.

## Open Questions

- **RQ-1** What does Postfix do when the primary destination accepts the connection but returns a 4xx rejection? (FR-1.12) Check `transport(5)` and `smtp(8)` for the packaged Postfix version, then update FR-1.12.
- **RQ-2** What does an upstream node do when it reads a transport line with two destinations during a mixed-version cluster upgrade? (FR-3.3) Check by testing, or avoid by upgrading all nodes before setting a backup host.
- **RQ-3** Does the web GUI (`pmg-gui`) need fork changes so administrators can set the backup host without `pmgsh`? Decide whether to fork `pmg-gui` too.
- **RQ-4** Does the API `comment` property accept a newline? (NFR-3.2, SC-004) Check `PVE::JSONSchema` string handling and test.

Delegated decisions so far: FR-1.3 and FR-1.4.

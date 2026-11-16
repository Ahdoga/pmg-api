# Security: pmg-api fork

This file owns the secure-coding rules for the fork. `REQUIREMENTS.md` owns WHAT the fork does, `DESIGN.md` owns API and command-line conventions, and `ARCHITECTURE.md` owns components, stack, and trust boundaries (TB-1 to TB-5). This file contains no threat model or risk scoring.

## Secure-Coding Inputs

**Documents read (2026-11-16):** `REQUIREMENTS.md`, `ARCHITECTURE.md`, `DESIGN.md`, and the upstream code in `src/PMG/API2/Transport.pm`, `src/PMG/Config.pm` (transport functions), `src/PMG/Utils.pm` (`run_postmap`), `src/PMG/Cluster.pm`, and `debian/control`. No earlier `SECURITY.md` existed.

**Security notes from the owner:** none. The fork is personal and is deployed on the owner's own mail gateway.

**Confirmed components (from ARCHITECTURE.md):** Perl; `PVE::RESTHandler`, `PVE::JSONSchema`, `PVE::INotify`, `PVE::Tools`; Template Toolkit; Postfix `>= 3.7.10`; the `postmap` program; flat files under `/etc/pmg`.

**Applicability decisions:**
- *API access control and input validation:* applicable. A mail gateway sits on the path of all mail, so a wrong transport entry can divert mail.
- *Configuration file integrity:* applicable. Values are written into a Postfix map file (NFR-3.2).
- *Running external programs:* applicable (`postmap`).
- *Output encoding, CSRF, sessions, password storage, CORS:* not applicable in this repository. Upstream code owns them (`PMG::AccessControl`, `pmgproxy`), and the fork changes none of it.
- *Secrets and cryptography:* not applicable. The fork adds no secret and no cryptography.
- *Database access:* not applicable. The fork adds no SQL.
- *Outbound network requests:* not applicable. The fork adds none.
- *Dependencies and build:* applicable (supply chain for Debian packages).
- *LLM or AI-specific controls:* not applicable for the product. For the coding agent, see `AGENTS.md`.

**Gaps and conflicts:**
- `postmap` failures are reported only as a warning (`run_postmap` catches the error and calls `warn`). A failed rebuild leaves the old lookup table in use while the API reports success. The fork does not change this (upstream behavior). Recorded as OD-001.
- It is not yet verified whether the API `comment` property can contain a newline, which would let a value add another line to `/etc/pmg/transport` when written as `#<comment>\n`. No rule depends on the answer, but SC-004 requires it to be checked (RQ-4).
- Industry standards (OWASP, NIST) have not been retrieved for this file. Rules rest on the code patterns named in their Evidence line. Add standard sources when a rule needs one.

## Sources

- **S-01 `src/PMG/API2/Transport.pm`** at commit `e889c13`, read 2026-11-16. Supports: SC-001, SC-002, SC-003.
- **S-02 `src/PMG/Config.pm`**, transport functions and `lock_config`, read 2026-11-16. Supports: SC-003, SC-004.
- **S-03 `src/PMG/Utils.pm`**, `run_postmap`, read 2026-11-16. Supports: SC-005.
- **S-04 `debian/control`** and `debian/copyright`, read 2026-11-16. Supports: SC-007, SC-008.
- **S-05 Proxmox forum, "Mail Problem"**, https://forum.proxmox.com/threads/mail-problem.73911/latest, quoting Postfix transport map multiple destinations (Postfix 3.5 or later). Secondary source; confirm against `transport(5)` for the packaged Postfix version.

## Secure-Coding Rules

### API access control and input validation

**SC-001 — Every API method declares its permissions**
- **Status:** ACTIVE
- **Rule:** Every `register_method` call MUST set `permissions`. A method that changes state MUST set `protected => 1`, MUST allow only the `admin` role, and, for cluster-wide configuration, MUST set `proxyto => 'master'`. A new method MUST NOT be reachable without authentication.
- **Enforcement:** `PMG::API2::*` modules.
- **Evidence:** NFR-3.1; S-01.
- **Verification:** `pmgsh verifyapi` passes. A request without a ticket, and a request from an `audit` user to a write method, are both refused.

**SC-002 — Every parameter has a type and a format**
- **Status:** ACTIVE
- **Rule:** Every API parameter MUST declare a `type`. Host, domain, and address values MUST also declare a registered `format`. A method MUST set `additionalProperties => 0`.
- **Enforcement:** `parameters` block of each method.
- **Evidence:** NFR-3.2; FR-2.3; S-01.
- **Verification:** A request with an unknown parameter, a non-address `backup_host`, or a `backup_port` of 0 or 70000 is refused and changes nothing.

### Configuration files

**SC-003 — Locked, registered file writes**
- **Status:** ACTIVE
- **Rule:** Code MUST change files under `/etc/pmg` only through `PVE::INotify::write_file` on a registered file, inside `PMG::Config::lock_config`. It MUST NOT open those files for writing any other way.
- **Enforcement:** `PMG::Config`; API modules.
- **Evidence:** DR-4; TB-2; S-01; S-02.
- **Verification:** Two simultaneous create requests for different domains both appear in the final file, and no line is lost or merged.

**SC-004 — Values written to a Postfix map are validated**
- **Status:** ACTIVE
- **Rule:** Each value written into a Postfix map file MUST first pass a registered format that forbids whitespace, commas, and line breaks (for hosts and domains). A free-text value, such as a comment, MUST be written only on its own comment line and MUST NOT contain a line break.
- **Enforcement:** `PMG::Config` formats and `write_transport_map`.
- **Evidence:** NFR-3.2; TB-2; S-02. Open: RQ-4.
- **Verification:** A comment containing `\nevil.example smtp:[10.0.0.9]:25` is refused or stored on one line, and the file has no new entry. A host with a comma is refused.

**SC-005 — External programs run as a list**
- **Status:** ACTIVE
- **Rule:** External programs MUST run through `PVE::Tools::run_command` with an array of arguments and a fixed absolute program path. Code MUST NOT pass a string to a shell or build a command line from request data.
- **Enforcement:** `PMG::Utils::run_postmap` and any new caller.
- **Evidence:** DR-5; TB-3; S-03.
- **Verification:** A search of the diff finds no backticks, `system` with one string, or `open` with a pipe.

**SC-006 — Reads tolerate bad data**
- **Status:** ACTIVE
- **Rule:** Parsers for files in `/etc/pmg` MUST log a warning and skip a line they cannot parse. They MUST NOT die, and MUST NOT guess a value.
- **Enforcement:** `read_transport_map` and similar readers.
- **Evidence:** FR-1.8; NFR-2.3; S-02.
- **Verification:** A file with a three-destination line, a garbled line, and one good line loads the good entry and logs two warnings.

### Dependencies, build, and licensing

**SC-007 — No new dependencies without a decision**
- **Status:** ACTIVE
- **Rule:** A change MUST NOT add to `Build-Depends` or `Depends` in `debian/control`, or load a new Perl module, unless a decision is recorded in `ARCHITECTURE.md` (Stack table) in the same commit. Package names and versions MUST come from the Debian or Proxmox repository, checked live, and not from memory.
- **Enforcement:** Review; `debian/control`.
- **Evidence:** NFR-2.2; S-04.
- **Verification:** `git diff master -- debian/control` is empty unless the Stack table changed.

**SC-008 — Licenses and notices stay**
- **Status:** ACTIVE
- **Rule:** Changes MUST NOT remove or alter copyright or license notices in `debian/copyright` or in source files. New files that carry code MUST use the same AGPL v3-or-later notice as the files they sit beside.
- **Enforcement:** Review.
- **Evidence:** FR-4.3; NFR-1.1; S-04.
- **Verification:** `git diff master -- debian/copyright` is empty. Header lines of changed files are unchanged.

**SC-009 — Verified build before install**
- **Status:** ACTIVE
- **Rule:** A package MUST be built with `make deb` and pass `lintian`, and `pmgsh verifyapi` MUST pass, before it is installed on a production gateway. The package MUST be tested first on a non-production PMG VM.
- **Enforcement:** Owner.
- **Evidence:** NFR-4.3; FR-2.5; ARCHITECTURE.md Toolchain.
- **Verification:** The build log and `lintian` output are saved with the change.

## Open Security Decisions

- **OD-001** `postmap` errors only produce a warning, so the API can report success while Postfix keeps the old table. Decide whether the fork should fail the request on a `postmap` error. This is an upstream behavior, so a change should be proposed upstream first.
- **OD-002** RQ-4: confirm how `comment` is validated. If it can hold a line break, fix it and send the fix upstream.

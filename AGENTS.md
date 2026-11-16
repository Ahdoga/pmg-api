# Agent Instructions

@REQUIREMENTS.md
@ARCHITECTURE.md
@SECURITY.md
@DESIGN.md

This repository is a personal fork of `proxmox/pmg-api` (Proxmox Mail Gateway API, Perl, Debian package). It adds a backup (failover) destination to transport map entries. Everything else is upstream code and should stay as it is.

## Working rules

- Before writing code in an area, read every `SC-*` rule in `SECURITY.md` that names the component, and the code in `src/PMG` that the rule cites.
- Every leaf implementation issue follows `REQUIREMENT_TEMPLATE.md`. Epic, workstream, and decision issues use short, purpose-specific bodies that link the affected spec IDs and issues.
- A change that alters behavior, a component, the stack, or a security rule updates the owning spec in the same pull request.
- **Stay close to upstream.** Change the smallest set of lines that satisfies the requirement. Do not reformat, rename, reorder, or tidy code you are not changing (FR-4.2). Make one logical change per commit, with a message in the upstream style (`area: short imperative summary`, then a body that says why).
- Fork-only files (`AGENTS.md`, `REQUIREMENTS.md`, `ARCHITECTURE.md`, `SECURITY.md`, `DESIGN.md`, `REQUIREMENT_TEMPLATE.md`, `README.md`) must never be part of a patch meant for upstream (FR-4.4).
- Do not invent requirements. If a task needs behavior that no `FR-*` or `NFR-*` states, stop and ask, or propose the requirement in `REQUIREMENTS.md` first.
- When a spec says `TO BE DECIDED`, `UNKNOWN`, or lists an open question (RQ, AQ, OD, DQ), do not guess. Ask, then record the answer in the owning spec.
- Match the surrounding Perl style (`use strict; use warnings;`, 4-space indent, `die "...\n"` messages). Run `make tidy` before committing when it is available.
- Do not add Perl modules, Debian packages, or `debian/control` entries without a recorded decision (SC-007).
- Never edit `debian/copyright` or license headers (SC-008).
- Do not mark a task done until the applicable checks in the Commands table pass. The Claude workspace lacks the Proxmox Perl modules, so most checks cannot run there. Say plainly which checks ran and which did not; do not claim a test passed that was not run.
- A change that was only tested with stub modules MUST be called "unverified on PMG" in the commit or pull request, until it runs on a PMG test VM.
- This package runs on a mail gateway that handles all mail. Never apply an untested change to a production gateway (SC-009).

## Commands

Keep identical to the Toolchain table in `ARCHITECTURE.md`. These need a Proxmox or Debian trixie machine with the Proxmox repositories.

| Check | Command | Notes |
|---|---|---|
| Install build dependencies | `apt build-dep pmg-api` or `mk-build-deps` on `debian/control` | Needs the Proxmox repository |
| Format | `make tidy` | `proxmox-perltidy` |
| API check | `perl -I src src/bin/pmgsh verifyapi` | Also run by `make -C src install` |
| Tests | `make -C src/tests check` | Integration tests skipped in `sbuild` |
| Build package | `make deb` | Then `lintian` runs |
| Syntax check | `perl -I src -c <module>.pm` | Needs PVE modules installed |

## Third-party libraries

- Minimize third-party dependencies to reduce ongoing updates and maintenance. Prefer Perl core modules and the `PVE::*` helpers already used in the code.
- Add a library only when its benefits justify its ongoing maintenance, security, and upgrade costs. For simple, narrowly scoped functionality, a small local function is usually easier to verify. For security-sensitive functionality, use established libraries or existing helpers (`PVE::Tools::run_command`, `PVE::JSONSchema`) rather than hand-rolling replacements.
- Before adding a library, verify the exact package name and publisher against the Debian or Proxmox repository, and check its entry live. Never select package names or versions from model memory. Investigate lookalike names and unexpected ownership changes.
- Check the license of the package and its dependencies against the AGPL v3-or-later license of this project. Flag unclear or incompatible terms for the owner before proceeding.
- Assess maintenance through support status, unresolved issues, and security responsiveness. Reject abandoned packages.
- Check current security advisories (Debian Security Tracker and the upstream project) and report the sources and date checked. If advisory lookup is unavailable, stop and report that limitation; do not describe an unchecked version as vulnerability-free.
- Keep updates scoped to the task. Do not make unrelated upgrades.
- If you cannot verify these requirements, stop and report the blocker. Do not add the dependency unless the owner explicitly accepts the identified risk.

### Project application

- The only approved stack is the table in `ARCHITECTURE.md`. A package outside it needs a new decision recorded there before it is added (SC-007).
- Record the decision, the package, the version, and the date checked in `ARCHITECTURE.md` in the same commit that adds the dependency.

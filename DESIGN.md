# Design: pmg-api fork

This file owns how the fork's interfaces look and behave: API shape, command-line behavior, messages, and defaults. `REQUIREMENTS.md` owns WHAT the fork does, `ARCHITECTURE.md` owns components, and `SECURITY.md` owns controls.

`pmg-api` has no screens. The web GUI is the separate `pmg-gui` repository (RQ-3). If the fork gains GUI changes, add visual sections here and follow the Proxmox web interface look instead of inventing a new one.

## Overview

**Brief.**
- **Subject:** the REST API and `pmgsh` command line of a mail gateway.
- **Audience:** administrators who run a PMG node, often through the GUI or scripts.
- **Primary job:** let an administrator name a second mail server for a domain, and see that choice clearly in every read.

**Rationale.** Administrators already know the transport fields. The fork adds fields that look and behave like the existing ones, so there is nothing new to learn.

## API conventions

- **Naming.** Property names are lowercase with underscores. The backup fields mirror the primary ones with a `backup_` prefix: `backup_host`, `backup_port`, `backup_use_mx`.
- **Optional by default.** New properties are `optional => 1`. A request that omits them behaves as before (FR-2.4).
- **Defaults.** `backup_port` defaults to the primary `port`, and `backup_use_mx` defaults to the primary `use_mx` (FR-1.3, FR-1.4). The default applies when the backup host is created. The API schema describes this in the property description, and does not use a `default` key, because the default depends on another property.
- **Clearing a value.** An empty string for `backup_host` on update removes the whole backup destination (FR-1.6). Other fields cannot be cleared this way.
- **Read responses.** Backup fields appear only when set (FR-2.1). Clients must treat their absence as "no backup".
- **Descriptions.** Every property has a one-sentence description that says what it does and any limit (for example "Only supported for the 'smtp' protocol").
- **Formats.** Host-like values use the registered `transport-address` format. A property that also allows the empty string uses `transport-address-or-empty`.

## Messages

- Errors are one plain sentence ending with a line break, in the style of upstream (`die "Transport map entry '$domain' does not exist\n"`).
- Say what is wrong and which value, not how the code failed: `backup host is only supported for the smtp protocol`.
- Parse warnings for `/etc/pmg/transport` name the file, the line, and the reason, as upstream does.
- Never print secrets, tickets, or keys in a message.

## File format

- One entry per line: `<domain> <protocol>:<destination>[, <destination>]`.
- A comment line `#<text>` directly above an entry is that entry's comment.
- A destination is `host:port`. A host is wrapped in square brackets when MX lookups are off, or when it is an IP address.
- A second destination is the backup, written after a comma and a space.
- The file stays readable and editable by hand, and a hand edit that follows these rules loads without errors.

## Command line

- `pmgsh` follows the API: `pmgsh create /config/transport --domain example.com --host 10.0.0.1 --backup_host 10.0.0.2`, `pmgsh set /config/transport/example.com --backup_host ''`, `pmgsh get /config/transport`.
- Help text and bash completion come from the API schema, so descriptions in the schema are the documentation (FR-2.6).

## Do's and Don'ts

- Do keep changes small enough to send upstream as one patch (FR-4.1).
- Do match the style and wording of the surrounding upstream code.
- Do write the property description before the code.
- Don't rename or reformat upstream code that is not part of the change (FR-4.2).
- Don't add a property without a schema description.
- Don't change an existing property's meaning, default, or type.

## Open Questions

- **DQ-1** Should the GUI show the backup host as a separate field or as a second row? Decide with RQ-3.

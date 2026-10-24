# Roadmap

This roadmap covers the TiG standard and its reference implementation (`tig-lib`). Product roadmaps that use TiG, such as GTdb in DriversClub, are tracked separately.

## Why prototype with devices?

The first prototypes record the history of **computing devices**: desktops, servers and mobile phones. This domain was chosen because:

- **Uncontroversial data.** Hardware specs, OS updates and disk usage carry little personal or regulatory risk, unlike medical or even vehicle data.
- **Plenty of data, no manual entry.** Devices can report on themselves, so realistic timelines build up quickly without anyone typing in receipts.
- **Covers every entity type.** Each of the four TiG entities has a natural fit (see below), so the prototypes test the whole model rather than one part of it.
- **Developers already have the hardware.** Contributors can run a prototype on their own machines on day one.
- **Real ownership transfers.** Machines get sold, handed down, reimaged and decommissioned, which gives a low-stakes test of transferring a repository.

| Entity | Desktop / Server | Mobile phone |
| :---- | :---- | :---- |
| State | CPU, RAM, disks, OS, installed packages | Model, storage, OS version, installed apps |
| Events | OS upgrade, package install/remove, disk replaced | OS update, app installed, screen or battery replaced |
| Telemetry | Disk usage, uptime, CPU temperature | Battery health, storage used |
| Evidence | Purchase invoice, warranty certificate | Receipt, repair shop invoice |

## Components

| Name | Role |
| :---- | :---- |
| `tig-agent` | Collector that runs on each device and records state, events and telemetry into the device's TiG repository. It generates and keeps its own device key, and signs every commit with it. On desktops and servers it runs as the `tig-agentd` background service; on phones it is an app. |
| TiG Notary (`tig-notary`) | Server that signs commit hashes it receives from agents by returning signed Git tags. Agents authenticate with their device key, so there are no accounts or API keys, acting as the `Issuer` in `COMMITS.md`. It attests when a commit existed, not whether its contents are true. It stores signatures and keys, not repositories. |
| Remote | Any Git remote. Hosted TiG storage is deliberately out of scope for the prototypes (see open questions). |

## Phases

### 0. Spec foundations

- Repository layout, as described in the whitepaper (`/state`, `/events`, `/telemetry`, `/evidence`, git-ignored `/summaries`).
- File naming and formats for each entity type (see `lola-test` for an early YAML event example).
- Commit signing v1: signed tags (`COMMITS.md`).
- Telemetry streams, chunks and verification (`TELEMETRY.md`, draft).
- `tig-lib`: create, open and clone a repository; write and read entities; store and verify signature tags.

### 1. Desktop and server prototype

- `tig-agentd` (Linux first): a small agent that takes a snapshot of initial state, records changes as events and appends telemetry on a schedule.
- Rebuild current state by replaying events, and check it against a fresh snapshot.
- TiG Notary: build and sign tags for commit hashes sent by agents, and publish its allowed signers file at `/.well-known/tig-public-key`.
- Verify signatures across a repository's history.
- Sample repositories and tests generated from real machines.

### 2. Mobile prototype

- `tig-agent` for phones: record state and events through `tig-lib` bindings on iOS and Android, which tests how portable the C library really is.
- Sync between a device and a remote the owner provides (e.g. their own server or an existing Git host) with plain Git push and pull.

### 3. Transfer and sharing

- Specification for transferring ownership, tried out by selling or handing over a prototype device.
- Sharing a subset of history with a third party (e.g. a repair shop sees only hardware events).
- Encrypting a subset of history.

### 4. First production vertical: automotive (GTdb)

- Apply the proven layout and library to vehicles, branded as GTdb, starting with DriversClub.

### Future verticals

Housing, then pets, then health, each once the previous verticals have built trust in the format.

## Open questions

- ~~**Who signs?**~~ Decided: agents send commit hashes to TiG Notary, which returns signed Git tags (see `COMMITS.md`).
- ~~**Identity?**~~ Decided for now: devices generate their own pseudonymous keys, with no accounts. Stronger identities (verified owners, organisations such as garages) may be added later, where a use case needs them.
- **Device key lifecycle.** What happens when a device is reimaged and loses its key, when a key is rotated (e.g. the old key signs a commit adding the new one), and when the device is sold.
- **Notary abuse.** With no accounts, anyone can generate a key and request signatures. Rate limiting per key and per IP address is probably enough for the prototype.
- **Hosted storage needs a business plan.** Hosting repositories, especially evidence files and long telemetry histories, could be expensive and complicated to run. Before offering hosted storage we need a business plan covering costs, who pays (owners, manufacturers, a premium tier), retention, and what happens to data if the service shuts down. Until then, TiG Notary signs hashes only, and owners bring their own remote.
- ~~**State or telemetry?**~~ Decided: telemetry is observed, events are done to the entity, so battery capacity is telemetry and replacing the battery is an event (see `TELEMETRY.md`).
- ~~**Telemetry volume.**~~ Decided: readings are grouped into append-only chunk files per stream, sized by a per-stream period (see `TELEMETRY.md`).

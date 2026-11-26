# Roadmap

This roadmap covers the TiG standard and its reference implementation (`tig-lib`). Product roadmaps that use TiG are tracked separately. PC-db is a product, but it is also the first prototype of hosting, so the parts of it that touch the standard are covered here.

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
| Remote | Any Git remote. Hosted storage is out of scope for the prototypes, except a TiG host that its owner runs themselves (see below and the open questions). |
| TiG host | A service that hosts repositories for accounts. It speaks Git over HTTPS and checks every push against the TiG rules: fast-forward only, commits signed by listed keys, and only TiG paths. It can act as a *custodian* (see `COMMITS.md`), so that a device that loses its key can be re-enrolled, and it moves a repository between accounts when a device is sold. Accounts authorise hosting only: commits are still identified by device keys. Open source and self-hostable. |
| PC-db client | The Windows client for PC gamers: a service that records a PC's hardware, operating system and graphics driver history (`vocabularies/DEVICE.md`), and a tray app. It works with no server, with any Git remote, or with a TiG host. Its first job is to exercise the TiG host. |

## Phases

### 0. Spec foundations

- Repository layout, as described in the whitepaper (`/state`, `/events`, `/telemetry`, `/evidence`, git-ignored `/summaries`).
- File naming and formats for each entity type.
- Commit signing v1: signed tags (`COMMITS.md`).
- Telemetry streams, chunks and verification (`TELEMETRY.md`, draft).
- Events and state: event files, changes, replay and corrections (`EVENTS.md`, draft).
- `tig-lib`: create, open and clone a repository; write and read entities; store and verify signature tags.

### 1. Desktop and server prototype

- `tig-agentd` (Linux first): a small agent that takes a snapshot of initial state, records changes as events and appends telemetry on a schedule.
- Rebuild current state by replaying events, and check it against a fresh snapshot.
- TiG Notary: build and sign tags for commit hashes sent by agents, and publish its allowed signers file at `/.well-known/tig-public-key`.
- Verify signatures across a repository's history.
- Sample repositories and tests generated from real machines.

### 1b. Windows PCs and hosting (PC-db)

A prototype of hosting, key succession and transfer, using Windows gaming PCs. PC-db's own roadmap is kept with the product; this is the
part that touches the standard.

- Additions to the device vocabulary (`vocabularies/DEVICE.md`): graphics adapters and drivers, Windows sources, and opt-in game and
  benchmark summaries. They are platform-neutral, so a Linux agent can record them too, and macOS, Android and single-board computers
  can follow.
- A Windows client, a service and a tray app built on `tig-lib`, that records a PC's history with no server, with any Git remote, or with a
  TiG host.
- A TiG host: accounts (email sign-in first), Git over HTTPS with every push checked against the TiG rules, and a custodian that
  re-enrols a device whose key was lost in a reset (`COMMITS.md`, "Custodians").
- Adopting a host from an existing repository, and leaving it again, without changing the repository's history.
- Moving a repository from one account to another when the PC is sold.

### 2. Mobile prototype

- `tig-agent` for phones: record state and events through `tig-lib` bindings on iOS and Android, which tests how portable the C library really is.
- Sync between a device and a remote the owner provides (e.g. their own server or an existing Git host) with plain Git push and pull.

### 3. Transfer and sharing

- Specification for transferring ownership, tried out by selling or handing over a prototype device. PC-db's host (phase 1b) tries a transfer between accounts first.
- Sharing a subset of history with a third party (e.g. a repair shop sees only hardware events).
- Encrypting a subset of history.

### 4. First production vertical: automotive

- Apply the proven layout and library to vehicles.

### Future verticals

Housing, then pets, then health, each once the previous verticals have built trust in the format.

## Open questions

- ~~**Who signs?**~~ Decided: agents send commit hashes to TiG Notary, which returns signed Git tags (see `COMMITS.md`).
- ~~**Identity?**~~ Decided for now: devices generate their own pseudonymous keys, with no accounts. Stronger identities (verified owners, organisations such as garages) may be added later, where a use case needs them. A TiG host has accounts, but only to authorise hosting and transfer: commits are still identified by device keys, and an account's email address is never recorded in a repository.
- **Device key lifecycle.** Partly decided: a hosted repository can list a *custodian* that re-enrols a device after it is reimaged and loses its key (see `COMMITS.md`, "Custodians"), and a device that still has its key can rotate it (the old key signs a commit adding the new one, see "Several signers"). A repository that isn't hosted, whose device loses its key, starts a new one. Still open: handing a device on when it is sold (a host can do it for a hosted repository, see phase 1b), and revocation. Signers can be added but never removed, so a lost, compromised or no longer trusted key (e.g. a previous owner's, or a garage the owner stops using) needs a way to be revoked.
- **Custodian scope.** A custodian can add a key to every repository that lists it, and an added key can then write new events, though never change history that a notary signature already covers. A key per account or per repository would limit the harm if a host is compromised, but each key must then be kept safe. How custodian keys are rotated is also unspecified.
- **Notary abuse.** With no accounts, anyone can generate a key and request signatures. Rate limiting per key and per IP address is probably enough for the prototype.
- **Hosted storage needs a business plan.** Hosting repositories, especially evidence files and long telemetry histories, could be expensive and complicated to run. A TiG host run by the person who uses it, as in the PC-db prototype, doesn't raise these questions, since nobody else depends on it. Before offering hosted storage to others we need a business plan covering costs, who pays (owners, manufacturers, a premium tier), retention, and what happens to data if the service shuts down. Until then, TiG Notary signs hashes only, and owners bring their own remote or run their own host.
- ~~**State or telemetry?**~~ Decided: telemetry is observed, events are done to the entity, so battery capacity is telemetry and replacing the battery is an event (see `TELEMETRY.md`).
- ~~**Telemetry volume.**~~ Decided: readings are grouped into append-only chunk files per stream, sized by a per-stream period (see `TELEMETRY.md`).

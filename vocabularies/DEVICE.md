## Device vocabulary (`tig-device/1`)

This vocabulary describes computing devices, such as desktops, servers, laptops, phones and single-board computers, as recorded by their own agent (e.g. `tig-agent`). It
adds the meaning that `EVENTS.md` leaves to vocabularies: entity attributes, item kinds, event and action types, and stream names. It is a
draft: [Open questions](#open-questions) lists what is still undecided.

The vocabulary is platform-neutral: every attribute, item and stream in it is meant to be recordable on any platform, though a platform
may lack some (Windows has no load average, and macOS drivers ship with the operating system). [Linux sources](#linux-sources) and
[Windows sources](#windows-sources) say where an agent on each finds each value; agents for other platforms, such as macOS, Android and
single-board computers, map their own.

### What is recorded

> Record what describes the device; leave out what describes its owner. Ask: would a future owner mind seeing it?

A device's repository goes with the device when it is sold or handed on, so it is written for the next owner as much as the current one.

| Recorded | Not recorded |
| --- | --- |
| Hardware identity: model and serial numbers, MAC addresses | IP addresses, network names (e.g. Wi-Fi SSIDs), VPN configuration |
| Components: firmware, CPUs, memory, disks, graphics adapters and drivers, network adapters, batteries | Hostname, which the owner chooses and the next owner will change |
| Filesystems on internal disks, the operating system and kernel | User accounts, home directories, files, product keys, licence status |
| Boots | Network shares, removable media, other software |
| Docker Compose stacks and their containers, if the owner turns this on ([Compose stacks](#compose-stacks)) | A container's environment, ports, volumes, networks and secrets |
| Play and benchmark summaries, if the owner turns them on ([Play and benchmarks](#play-and-benchmarks)) | Window titles, file paths, command lines, server addresses, anything typed or shown in a game |

Hardware identity is the device's equivalent of a car's VIN. Repositories are not generally public, but they may be shown to a buyer.

Compose stacks and play are the two things here that describe what the device is *used for*. Stacks are recorded because a server's history
is largely its services' history, and play because a gaming PC's is largely how its games ran. Both are recorded only when the owner enables
them, and only what identifies a service and its image, or a title and how it ran, never how either is configured.

### Signers

A device's repository is written by the device itself, so it lists only the device's own keys. A device that loses its key, for instance
when it is reset, makes a new one. If the repository is hosted, a *custodian* adds it (see `COMMITS.md`, "Custodians"), so the repository
may list several keys of the same device, one after another, and a custodian. A verifier reports any other signer, since it means
something other than the device wrote to the repository. Without a custodian, a device that loses its key starts a new repository.

### Entity and initial state

The agent writes `state/initial.yaml` when it creates the repository. It holds only the entity, whose attributes describe the device as
built. Components are added by a [discovery](#discovery) event, because they may have been replaced before the agent first ran (see
`EVENTS.md`, "Initial state").

```yaml
# state/initial.yaml
kind: device
vocabulary: tig-device/1
items:
  entity:
    vendor: Example Computers
    model: Mini PC X1
    serial: "EXC1234567"
    architecture: x86_64
```

| Attribute | Description | Format |
| --- | --- | --- |
| `vendor` | Manufacturer of the device | Text |
| `model` | Product name | Text |
| `serial` | Serial number of the device | Text |
| `architecture` | Instruction set | Text, e.g. `x86_64`, `aarch64` |
| `virtualisation` | The hypervisor, for a virtual machine; absent for physical hardware | Text, e.g. `kvm`, `vmware` |
| `form-factor` | The kind of device, if the firmware or operating system reports it | `desktop`, `laptop`, `all-in-one`, `server`, `handheld`, `phone`, `tablet`, `single-board` or `other` |

Attributes whose value is unknown or a placeholder (e.g. `To Be Filled By O.E.M.`) are left out.

### Items

| Kind | ID | Attributes |
| --- | --- | --- |
| `firmware` | `firmware` | `vendor`, `version`, `date` |
| `cpu` | `cpu/<socket>`, e.g. `cpu/0` | `model`, `cores`, `threads` |
| `memory` | `memory` | `capacity` |
| `disk` | `disk/<hash>` | `model`, `serial`, `wwn`, `capacity`, `transport`, `rotational` |
| `filesystem` | `filesystem/<mount>` | `type`, `uuid`, `size`, `disks` |
| `network` | `network/<hash>` | `mac`, `medium`, `driver`, `hardware` |
| `gpu` | `gpu/<hash>` | `vendor`, `model`, `memory`, `hardware` |
| `driver` | `driver/gpu/<hash>` | `provider`, `version`, `date` |
| `os` | `os` | `id`, `version`, `name`, `kernel` |
| `battery` | `battery/<hash>` or `battery/<name>` | `manufacturer`, `model`, `serial`, `technology`, `design-energy`, `design-charge` |
| `stack` | `stack/<project>` | `project` |
| `container` | `container/<project>/<service>/<n>` | `image`, `digest`, `created` |

Sizes and capacities are in bytes, `design-energy` is in watt-hours and `design-charge` in amp-hours. Dates are `YYYY-MM-DD`. An attribute
the device doesn't report is left out.

**Hashed IDs.** A component that has its own identity, such as a disk's serial number, gets an ID from it: the kind, `/`, then the first 8
hex digits (lowercase) of the SHA-256 of the identity, as UTF-8, with its parts joined by a newline (`\n`). Hashing keeps the ID within
the characters an item ID allows, and a component with its own identity gets the same ID in every repository, which may later link a
component moved between devices (see `EVENTS.md`, "Items moving between entities"). The identity itself is kept in an attribute.

| Kind | Identity hashed | If there isn't one |
| --- | --- | --- |
| `disk` | `<model>\n<serial>`, trimmed | The WWN; then `path:<bus path>`, e.g. `path:pci-0000:00:04.0` |
| `network` | The permanent MAC address, lowercase, colon-separated | Not recorded (see below) |
| `battery` | `<manufacturer>\n<model>\n<serial>`, trimmed | The role, e.g. `battery/bat0` |
| `gpu` | `<vendor ID>:<device ID>:<subsystem ID>\n<bus location>` (see [Graphics adapters](#graphics-adapters)) | The model name, for an adapter that is part of the processor and has no bus location |

A component without its own identity is told apart only by where it is. Moving it, or replacing it with an identical one, can't be
detected, or looks like a replacement.

#### Firmware

The system firmware (BIOS or UEFI). `version` and `date` are as the firmware reports them. An update is `upgraded`.

#### CPUs

One item per populated socket. `model` is the name the processor reports, `cores` its physical cores and `threads` its hardware threads.
CPUs have no readable serial number, so a CPU swapped for an identical one is not detected.

#### Memory

`capacity` is the physical memory installed, not the memory the operating system makes available, which varies between kernels. A
virtual machine's memory may be resized while it runs.

#### Disks

Internal, non-removable block devices: whole disks, not partitions. `transport` is `nvme`, `sata`, `sas`, `mmc` or `virtio`, and
`rotational` is `true` for spinning disks. Removable media and USB-attached disks are not recorded, since they come and go; nor are
virtual devices (loop, RAM, swap, device-mapper and RAID devices), whose underlying disks are recorded instead.

#### Filesystems

Filesystems of a persistent type (e.g. `ext4`, `xfs`, `btrfs`, `f2fs`, `zfs`, `ntfs`, `refs`, `exfat`, `apfs`, and `vfat` for an EFI system
partition) on recorded disks. Pseudo, temporary, image and network filesystems (`proc`, `tmpfs`, `overlay`, `squashfs`, `nfs`, `cifs`, ...)
are not recorded, nor is one with neither a mount point nor a drive letter, such as Windows' recovery partition.

* The ID is the mount point, which is a role: the filesystem that holds the operating system (`/`, or the Windows system drive) is
  `filesystem/root`, and any other mount point is its path without the leading `/`, lowercased, with any character an item ID doesn't
  allow replaced by `-` (so `/boot/efi` is `filesystem/boot/efi`). Where filesystems have drive letters instead, the ID is the letter in
  lowercase, e.g. `filesystem/d`.
* A filesystem mounted at several points (e.g. by a bind mount) is recorded once, at its shortest mount point. Subvolumes mounted
  separately, such as btrfs `@` at `/` and `@home` at `/home`, are separate filesystems.
* `size` is the filesystem's total size. `disks` lists the IDs of the disks it is stored on, through any partitions, device-mapper
  (e.g. LVM, LUKS) or RAID layers.
* A filesystem recreated at the same mount point (a new `uuid`) is `replaced`: a `remove` and an `add` of the same ID.
* `uuid` is the filesystem's own unique ID: its UUID, or where it has none (Windows), its volume serial number in lowercase hexadecimal.
  Formatting a volume gives it a new one.

#### Network adapters

Physical network adapters only: virtual interfaces (bridges, VPN tunnels, container and loopback interfaces) are not recorded. `mac` is
the permanent address burnt into the adapter, not a randomised or assigned one, so privacy features such as MAC randomisation don't look
like a new adapter. An adapter that has no permanent address is not recorded. `medium` is `ethernet` or `wifi`, `driver` is the driver's
name, and `hardware` is the adapter's ID in its bus's scheme, e.g. `{ system: pci, value: "8086:15f3" }`.

#### Operating system

`id` and `version` identify the system and its release (e.g. `ubuntu`, `"24.04"`; `windows`, `"24H2"`; `macos`, `"15.1"`), and `name` is
its full name for people (e.g. `Ubuntu 24.04.1 LTS`, `Windows 11 Pro`). A change to any of them is `upgraded`, including point releases and
a change of edition. `kernel` is the running kernel's release (e.g. `6.8.0-48-generic`, or the full Windows version, `"10.0.26100.2314"`),
so it changes only at a boot (see [Boot](#boot)): a kernel update on Linux, or a Windows monthly update that changes nothing else, is
found that way.

#### Batteries

Batteries that power the device. `design-energy` or `design-charge`, whichever the battery reports, is its capacity when new. Its
current capacity is telemetry (see `TELEMETRY.md`, "Telemetry or state?").

#### Graphics adapters

Adapters that render graphics, integrated or discrete: a laptop may have two, and a phone or single-board computer usually has one that is
part of its processor. `vendor` is the maker, e.g. as named by the adapter's PCI vendor ID, and `model` is the name the driver or
firmware reports, e.g. `NVIDIA GeForce RTX 4070`. `memory` is the dedicated video memory in bytes; for an integrated adapter that is what
the firmware sets aside, and it is left out where the adapter just shares system memory. `hardware` is the adapter's ID in its bus's
scheme, as for network adapters, e.g. `{ system: pci, value: "10de:2786" }`, and is left out for an adapter that has no bus.

Most adapters have no serial number that can be read without the maker's tools, so one replaced by an identical card in the same slot
isn't detected, as for CPUs. Software renderers, such as Microsoft's basic render driver or Mesa's `llvmpipe`, aren't adapters and aren't
recorded.

The identity of an adapter with a PCI bus location is `<vendor ID>:<device ID>:<subsystem ID>\n<bus location>`. Its ID is a hash of that
text, so every agent writes it the same way: in lowercase hexadecimal, each number padded with zeroes to its full width.

| Part | Format | Value |
| --- | --- | --- |
| `<vendor ID>`, `<device ID>` | 4 digits each | As in `hardware` |
| `<subsystem ID>` | 8 digits | The subsystem ID followed by the subsystem vendor ID, which is the 32-bit value at offset `0x2c` of the adapter's configuration space. Zeroes where it has none. |
| `<bus location>` | `ssss:bb:dd.f` | The segment (4 digits), bus (2), device (2) and function (1), e.g. `0000:01:00.0` |

For example, an adapter `10de:2786` with subsystem ID `889d` and subsystem vendor ID `1043`, at bus 1, device 0, function 0 of segment 0,
has the identity `10de:2786:889d1043\n0000:01:00.0`, and so the ID `gpu/166f5f15`.

An adapter with no bus location is identified by its `model`, exactly as recorded.

#### Graphics drivers

A driver is its own item, so that replacing a card and updating its driver are different events. Its ID is `driver/gpu/` and the
adapter's hash. It is recorded only where the driver can change separately from the operating system: where drivers ship only with the
operating system, as on macOS, there is no `driver` item, and a change there is the operating system's `upgraded`.

* `provider` is the driver's publisher, `version` is the version the platform reports (e.g. `"32.0.15.6094"` on Windows, or `"560.35.03"`
  for NVIDIA's driver on Linux), and `date` is the driver's date, left out if the platform gives none.
* An adapter running the platform's generic driver (e.g. Windows' basic display driver) is still the same adapter: its driver item says so,
  and installing the maker's driver is `upgraded`.
* An adapter and its driver are added, removed and replaced together: `installed`, `removed` and `replaced` change both items, `resized`
  changes the adapter's `memory`, and `upgraded` changes the driver, including a roll-back to an older version.
* Where a driver has several parts with their own versions, such as a Linux kernel module and Mesa, which one `version` names is an
  [open question](#open-questions). So is the maker's own release number (the `560.94` on a download page), which can be derived from
  a Windows `version` by rules that differ between makers; version 1 doesn't record it.

#### Compose stacks

Optional: an agent records these only when the owner has enabled it and given the agent access to the Docker Engine API (read-only).

A **stack** is a Docker Compose project, and a **container** is one replica of one of its services. Both are identified by the labels
Compose puts on every container it creates (`com.docker.compose.project`, `.service` and `.container-number`), so containers started any
other way (`docker run`, Swarm, Kubernetes) are not recorded.

* A stack's ID is `stack/<project>` and a container's is `container/<project>/<service>/<n>`, where `<n>` is the replica number. Each part
  is lowercased and any character an item ID doesn't allow is replaced by `-`. The ID names a *role*, as for `filesystem`: a container
  recreated for the same service and replica is the same item.
* `project` is the Compose project name as Compose reports it.
* `image` is the image reference the service is configured with, without a registry credential (e.g. `postgres:16`). `digest` is the
  content ID of the image the container was created from (`sha256:...`), which is what changes when a mutable tag such as `latest` is
  pulled again. `created` is when the container was created, to the second.
* `version` is optional: the release the image's tag stands for, such as `19.5.5-ce.0` for `gitlab/gitlab-ce:latest`, when an agent is able
  to find it (the owner may allow it to ask Docker Hub for the tags that share the image's digest). It is absent when it can't be found, and an
  agent that can't look it up at some moment doesn't remove one already recorded for the same image. A new `digest` or `image` with a new
  `version` is one `upgraded`. Labels inside the image are not a source, since they are inherited from its base image.
* A stack exists while any of its containers does. A container exists while Docker has it, running or not: a container that has stopped
  or crashed is still the item, and its state is telemetry. Stopped containers are therefore not `removed`, but `docker compose down` is.
* Recreating a container with a new `digest` or `image` is `upgraded`, which sets its new `created` too. Recreating it with the same image (e.g. changed configuration) is
  `reconfigured`, with `created` the only attribute to change.
* Containers started by `docker compose run` carry a `com.docker.compose.oneoff` label and are not recorded: they are tasks, not part of
  the stack.
* Not recorded: environment variables, command lines, ports, published addresses, volumes and bind mounts, networks, labels other than the
  three above, and the Compose file itself. These identify the owner's configuration and may contain secrets.

### Events

| Event type | Recorded | Actions |
| --- | --- | --- |
| `discovery` | When the agent first runs, and when a new agent version first records a kind of item | `discovered` |
| `boot` | Once per boot | `booted` |
| *(none)* | When the agent finds that a component, the operating system, a stack or a container has changed | `installed`, `removed`, `replaced`, `resized`, `upgraded`, `reconfigured` |

| Action type | Means | Changes |
| --- | --- | --- |
| `discovered` | Items found already present | `add`, each with `since` |
| `booted` | The device started | `update` of `os` `kernel`, only if it differs |
| `installed` | A component added | `add` |
| `removed` | A component taken out | `remove` |
| `replaced` | A component swapped for another, or a filesystem recreated | `remove` then `add`, and `update` of items that referred to the old one |
| `resized` | A capacity or size changed, e.g. memory added or a filesystem grown | `update` |
| `upgraded` | Firmware, the operating system or a graphics driver updated | `update` |
| `reconfigured` | Any other attribute changed, e.g. a virtual machine given more cores | `update` |

Every event sets `source` to the agent and its version, e.g. `tig-agent 0.6.0`. Updates set only the attributes that changed, and an
action is recorded only if something changed.

#### Discovery

The first discovery event is written when the agent creates the repository, at that time. It adds every item found, each with `since`: the
time it is known to have been present from (e.g. when a filesystem was created), or `unknown`. A later discovery event, written when a new
agent version starts recording a kind of item for the first time (e.g. batteries), adds only items of that kind.

#### Boot

`time` is when the device booted, to the second. The `booted` action has one extra field:

| Field | Description | Format |
| --- | --- | --- |
| `boot` | The boot's unique ID, from the operating system | Text: a UUID where the system gives one; otherwise the kernel's start time, in whole seconds, as an RFC 3339 UTC string |

The agent records each boot once, when it starts and finds that the boot's ID has no event yet. Where the ID is a start time, it treats start
times within a minute of each other as one boot, since the clock may have been corrected in between. The boot in progress when the agent first
runs happened before the discovery event, so its `booted` action has no changes.

#### Changes

The agent finds changes by comparing the device with state replayed to the current time. It does so when it starts and periodically
while it runs (e.g. at each commit), and may also react to the platform's device-change notifications, so that `time` is closer to when a
change happened. Each difference is one action, of the type in the table above; a component removed and another of the
same kind added in the same comparison is one `replaced` action.

A change is seen only some time after it happened, so the event has a `not_before`:

| Found | `time` | `not_before` |
| --- | --- | --- |
| At the first comparison after a boot | The boot time | The previous comparison, or the latest telemetry reading if it isn't known |
| At a later comparison | The time of the comparison | The previous comparison |

A change found at a boot is a separate event from the boot: the boot's time is exact, but the change happened at some point in the window
before it.

### Telemetry

Streams use these names, both in `/telemetry` and in an event's `readings`. Capacities and sizes are state, not telemetry, and the time
since boot is given by boot events.

| Stream | Columns | Unit |
| --- | --- | --- |
| `system.load` | `load1`, `load5`, `load15` | Run-queue length |
| `cpu.busy` | `busy` | Percent |
| `memory` | `available` | Bytes |
| `filesystem.<mount>` | `used`, `available` | Bytes |
| `battery.<id>` | `remaining`, `full`, `state` | Watt-hours or amp-hours; `state` is text |

`<mount>` is the filesystem's ID after `filesystem/`, with `/` replaced by `.`, e.g. `filesystem.root` or `filesystem.boot.efi`.

`system.load` is recorded only where the operating system has a load average, which Windows doesn't. `cpu.busy` is the percentage of all
logical processors that weren't idle since the previous reading, 0 to 100, and any platform can provide it. `<id>` is the battery's ID
after `battery/`, with `/` replaced by `.`. `remaining` and `full` are in the unit of the battery item's `design-energy` (watt-hours) or
`design-charge` (amp-hours), `full` being the charge it holds when full today, so `full` against the design capacity is its wear. `state`
is `charging`, `discharging` or `idle`. A device that sleeps or is off for long periods, such as a laptop or a phone, simply has gaps.

For [Compose stacks](#compose-stacks), one stream per container, sampled at the same rate as the rest:

| Stream | Columns | Unit |
| --- | --- | --- |
| `container.<project>.<service>.<n>` | `state`, `health`, `restarts`, `cpu`, `memory` | See below |

`<project>.<service>.<n>` is the container's ID after `container/`, with `/` replaced by `.`. `state` is the container's status as Docker
reports it (`running`, `restarting`, `exited`, `paused`, `dead`, `created`) and `health` is the result of its health check (`healthy`,
`unhealthy`, `starting`), empty if it has none. `restarts` is Docker's restart count for the container, which resets when it is recreated.
`cpu` is the CPU time used since the previous reading, in cores (1.0 is one core fully busy), and `memory` is the bytes in use, excluding
the page cache. `cpu` and `memory` are empty when the container isn't running. A container's stream is only defined once the container has
been seen, and a stream for a container that no longer exists simply stops.

#### Play and benchmarks

Optional, like [Compose stacks](#compose-stacks): an agent records these only when the owner has turned them on, and only the streams
below. What someone plays describes them rather than the device, so only what identifies the title and how it ran is recorded, never how
it was configured or what happened in it.

| Stream | Columns | Unit |
| --- | --- | --- |
| `play` | `app`, `seconds`, `fps`, `fps-low1`, `fps-low01`, `cpu`, `gpu`, `gpu-temp`, `gpu-power` | See below |
| `benchmark` | `suite`, `test`, `version`, `score`, `unit`, `seconds` | See below |

`play` has one reading for each stretch of play, with `time` the end of the stretch: a session, or each part of a long one.

| Column | Description | Unit |
| --- | --- | --- |
| `app` | The title: a store and its ID, e.g. `steam:1245620`; for anything else, the executable's or package's name, without its path | Text |
| `seconds` | How long the stretch lasted | Seconds |
| `fps` | Frames presented per second, on average | Frames per second |
| `fps-low1`, `fps-low01` | The average frame rate over the slowest 1% and 0.1% of frames | Frames per second |
| `cpu`, `gpu` | Average busy time of all logical processors, and of the graphics adapter's busiest engine | Percent |
| `gpu-temp`, `gpu-power` | Average temperature and power draw of the graphics adapter | Celsius, watts |

A value the device can't report is empty. Individual frames aren't recorded: they would dwarf the rest of the repository, and the summaries
answer what an owner asks.

`benchmark` has one reading for each run, with `time` when it finished. `suite` and `test` name the benchmark program and what it ran,
`version` is the program's version, and `score` is its result in `unit` (`points`, `fps`, ...). Whether a higher score is better is up
to the suite.

A reading records what the agent measured. The device key shows which device recorded it and a notary signature shows when it existed,
but neither shows that the result wasn't produced or altered some other way.

### Example

A server whose agent is installed in November 2026, then two boots. Before the second, its disk is cloned to a larger one and the root
filesystem grown, which the agent finds when it starts.

```yaml
# events/2026/2026-11-12.k3v9qa.yaml
time: "2026-11-12T09:30:02Z"
type: discovery
source: tig-agent 0.6.0
actions:
  - type: discovered
    changes:
      - add: firmware
        since: unknown
        set: { vendor: Example, version: "1.07", date: "2024-02-19" }
      - add: cpu/0
        since: unknown
        set: { model: Example CPU 8C, cores: 8, threads: 16 }
      - add: memory
        since: unknown
        set: { capacity: 34359738368 }
      - add: disk/7a858f1b
        since: unknown
        set: { model: Example NVMe 1TB, serial: "S6EXAMPLE001", capacity: 1000204886016, transport: nvme, rotational: false }
      - add: filesystem/root
        since: "2025-01-03T14:22:10Z"
        set: { type: ext4, uuid: 0b3e6c1d-5a7f-4c2e-9d18-6f4a2b7c9e01, size: 983349346304, disks: [disk/7a858f1b] }
      - add: network/87924582
        since: unknown
        set: { mac: "00:00:5e:00:53:01", medium: ethernet, driver: igc, hardware: { system: pci, value: "8086:15f3" } }
      - add: os
        since: "2025-01-03T14:22:10Z"
        set: { id: ubuntu, version: "24.04", name: Ubuntu 24.04.1 LTS, kernel: 6.8.0-48-generic }
```

```yaml
# events/2026/2026-11-10.x2m7pd.yaml: the boot in progress when the agent was installed
time: "2026-11-10T18:44:51Z"
type: boot
source: tig-agent 0.6.0
actions:
  - type: booted
    boot: 4d1b7a90-2c3e-4f6a-8b5d-9e0c1f2a3b4c
```

```yaml
# events/2026/2026-11-20.q9w3ne.yaml: found at the next boot
time: "2026-11-20T06:02:14Z"
not_before: "2026-11-19T22:41:00Z"
source: tig-agent 0.6.0
actions:
  - type: replaced
    changes:
      - remove: disk/7a858f1b
      - add: disk/eb3f1f45
        set: { model: Example NVMe 2TB, serial: "S7EXAMPLE002", capacity: 2000398934016, transport: nvme, rotational: false }
      - update: filesystem/root
        set: { disks: [disk/eb3f1f45] }
  - type: resized
    changes:
      - update: filesystem/root
        set: { size: 1967925690368 }
```

```yaml
# events/2026/2026-11-20.h8c4rv.yaml
time: "2026-11-20T06:02:14Z"
type: boot
source: tig-agent 0.6.0
actions:
  - type: booted
    boot: 8c2e4f1a-6b0d-4c55-9e3a-2f1d7c9b0a64
    changes:
      - update: os
        set: { kernel: 6.8.0-51-generic }
```

A gaming PC: a graphics driver updated, found at the next comparison an hour after the previous one.

```yaml
# events/2026/2026-11-18.c5d2yt.yaml
time: "2026-11-18T19:40:12Z"
not_before: "2026-11-18T18:40:12Z"
source: pc-db 0.1.0
actions:
  - type: upgraded
    changes:
      - update: driver/gpu/4be1c07a
        set: { version: "32.0.15.6094", date: "2024-10-02" }
```

A reading of `play`, which the owner has turned on, after half an hour of a game:

```
time,app,seconds,fps,fps-low1,fps-low01,cpu,gpu,gpu-temp,gpu-power
2026-11-18T21:30:00Z,steam:1245620,1800,96.4,71.2,58.9,34.0,92.5,68.0,201.4
```

### Linux sources

Where a Linux agent finds each value. All are readable by an unprivileged user except the device's serial number, which an agent that
runs unprivileged, as `tig-agent` does, reads once with privileges when it is installed (e.g. in the package's install script).
`tig-agent` doesn't yet record graphics adapters and drivers, `cpu.busy`, battery telemetry or play.

| Value | Source |
| --- | --- |
| `entity` `vendor`, `model`, `serial` | `/sys/class/dmi/id/sys_vendor`, `product_name`, `product_serial` (root only) |
| `entity` `architecture` | `uname()` machine |
| `entity` `virtualisation` | `systemd-detect-virt` |
| `entity` `form-factor` | `/sys/class/dmi/id/chassis_type` (the SMBIOS chassis type), where present |
| `firmware` | `/sys/class/dmi/id/bios_vendor`, `bios_version`, `bios_date` (`MM/DD/YYYY`) |
| `cpu` | `/proc/cpuinfo`: `physical id`, `model name`, `cpu cores`, `siblings` |
| `memory` `capacity` | `/sys/devices/system/memory/block_size_bytes` × the number of `memory*/online` that are `1`; else `MemTotal` in `/proc/meminfo`, rounded up to a GiB |
| `disk` | `/sys/block/<dev>`: `size` × 512, `removable`, `queue/rotational`; `ID_MODEL`, `ID_SERIAL_SHORT`, `ID_WWN`, `ID_BUS` and `ID_PATH` in `/run/udev/data/b<major>:<minor>` |
| `filesystem` | `/proc/self/mountinfo` (mount point, root, device, type); `statvfs()`; `/dev/disk/by-uuid`; disks through `/sys/dev/block/<major>:<minor>/slaves` |
| `network` | Interfaces in `/sys/class/net` that have a `device`; permanent MAC from `ETHTOOL_GPERMADDR`, or `address` where `addr_assign_type` is `0`; `wireless` or `phy80211` for Wi-Fi; `device/vendor` and `device/device`; `device/driver` |
| `os` | `ID`, `VERSION_ID` and `PRETTY_NAME` in `/etc/os-release`; `uname()` release |
| `battery` | `/sys/class/power_supply/<name>` where `type` is `Battery`: `manufacturer`, `model_name`, `serial_number`, `technology`, `energy_full_design` (µWh) or `charge_full_design` (µAh) |
| `gpu`, `driver` | `/sys/class/drm/card*/device`: `vendor`, `device`, `subsystem_vendor`, `subsystem_device` and the `driver` link; the bus location is the name of the directory that `device` links to; the driver's version from `/sys/module/<driver>/version` where it has one (NVIDIA's is in `/proc/driver/nvidia/version`) |
| Boot `time`, `boot` | `btime` in `/proc/stat`; `/proc/sys/kernel/random/boot_id` |
| `cpu.busy` | `/proc/stat`, as a difference between readings |
| `battery.<id>` | `/sys/class/power_supply/<name>`: `energy_now` and `energy_full` (µWh) or `charge_now` and `charge_full` (µAh), and `status` |
| `stack`, `container` | Docker Engine API `GET /containers/json?all=1`, keeping containers with a `com.docker.compose.project` label: `Labels`, `Image`, `ImageID`, `Created`, `State` |
| Container telemetry | `GET /containers/<id>/json` (`State.Health.Status`, `RestartCount`) and `GET /containers/<id>/stats?stream=false&one-shot=true` (`cpu_stats`, `precpu_stats`, `memory_stats`) |

### Windows sources

Where a Windows agent finds each value. These are indicative until an agent has implemented them, and which of them need more than a
standard service account is an open question.

| Value | Source |
| --- | --- |
| `entity` `vendor`, `model`, `serial`, `form-factor` | SMBIOS (`GetSystemFirmwareTable` with `RSMB`): the system and chassis structures |
| `entity` `architecture` | `GetNativeSystemInfo` |
| `entity` `virtualisation` | SMBIOS system manufacturer and product (e.g. `Microsoft Corporation`, `Virtual Machine`), not CPUID: a physical PC with virtualisation-based security also reports a hypervisor |
| `firmware` | SMBIOS BIOS structure: vendor, version and release date |
| `cpu` | SMBIOS processor structures; `GetLogicalProcessorInformationEx` for core and thread counts if SMBIOS lacks them |
| `memory` `capacity` | `GetPhysicallyInstalledSystemMemory` |
| `disk` | `IOCTL_STORAGE_QUERY_PROPERTY` (vendor, product, serial, bus type, device ID for the WWN, seek penalty for `rotational`) and `IOCTL_DISK_GET_LENGTH_INFO`; USB and removable disks are left out |
| `filesystem` | `GetVolumeInformation` (type, and the volume serial for `uuid`), `GetDiskFreeSpaceEx` (size), `IOCTL_VOLUME_GET_VOLUME_DISK_EXTENTS` (disks). The volume holding Windows is `filesystem/root`; the others are named by drive letter. |
| `network` | `GetAdaptersAddresses` (hardware interfaces of type Ethernet or Wi-Fi); the permanent address from the adapter (`OID_802_3_PERMANENT_ADDRESS`, `OID_802_11_PERMANENT_ADDRESS`) |
| `os` | `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion`: `DisplayVersion` for `version`; `ProductName` for `name`, corrected from the build number because it still says Windows 10 on Windows 11; and `10.0.<CurrentBuild>.<UBR>` for `kernel` |
| `battery` | `IOCTL_BATTERY_QUERY_INFORMATION` (manufacturer, name, serial, designed capacity) |
| `gpu`, `driver` | Display-class devices through Setup API. For `gpu`: the hardware ID, whose `SUBSYS_` field is the subsystem ID as written above; the segment and bus from `DEVPKEY_Device_BusNumber` (the segment taken to be the bits above the lowest 8, which is untested); and the device and function from the high and low 16 bits of `DEVPKEY_Device_Address`. `DEVPKEY_Device_DriverProvider`, `DEVPKEY_Device_DriverVersion` and `DEVPKEY_Device_DriverDate` for `driver`; `memory` from DXGI's `DedicatedVideoMemory` |
| Boot `time`, `boot` | The kernel's start time (`NtQuerySystemInformation` with `SystemTimeOfDayInformation`), which is also the boot's ID |
| `cpu.busy` | `GetSystemTimes`, as a difference between readings |
| `memory` stream | `GlobalMemoryStatusEx` |
| `filesystem.<mount>` | `GetDiskFreeSpaceEx` |
| `battery.<id>` | `IOCTL_BATTERY_QUERY_STATUS`, and the full-charge capacity from `IOCTL_BATTERY_QUERY_INFORMATION` |
| `play` | Present events from ETW (the DXGI and DxgKrnl providers, as tools such as PresentMon read them), the `GPU Engine` performance counters for busy time, and the adapter maker's API for temperature and power. An agent doesn't inject code into a game. |
| `benchmark` | The benchmark program's own result files or command line output, where it provides them |

### Open questions

* **Container crashes and restarts.** `restarts` and `state` in telemetry show that a container restarted, but not why (exit code, OOM kill).
  Is a crash something *done to* the device, and so an event, or an observation?
* **Compose privacy.** Service and image names reveal what a server is used for. Is opt-in enough, or should a repository be able to say
  that its stack data is stripped when the device is transferred?
* **Other container runtimes.** Podman, and containers not started by Compose, are not recorded.
* **Other writers.** Version 1 assumes the device's agent is its only writer, as `tig-agent` does. An owner adding a repair, with the
  shop's invoice as evidence, would be valuable. Should the vocabulary allow it, and which events may other signers record?
* **Packages.** Installed software is done to the device, but a Linux system has thousands of packages that update almost daily. If
  recorded, should it be one event per package-manager transaction, and only for packages the owner chose?
* **Shutdowns.** Whether a boot should record how the previous one ended (cleanly, or by a crash or power loss), as part of the device's
  reliability history.
* **More components.** Memory modules, displays and peripherals. Some need privileges to identify.
* **Per-action `not_before`.** A change found at a boot is a separate event so that the boot's time stays exact. Should `EVENTS.md` let an
  action narrow the event's time instead?
* **Reinstalls.** A reinstalled operating system loses the device key. A hosted repository carries on, because a custodian adds the new
  key (see [Signers](#signers)); one that isn't hosted starts a new repository (see `ROADMAP.md`, "Device key lifecycle"). Should that
  new repository refer to the old one, and how would a verifier know the claim is genuine?
* **Other platforms.** macOS, Android and single-board computers need their own sources. Things to settle: drivers that ship only with
  the operating system (so no `driver` item), graphics and memory that are part of one system-on-chip, and fixed storage that reports no
  model or serial number.
* **Boot time as an ID.** Where the system gives no boot ID, whether the kernel's start time is steady enough to identify a boot, and the
  right tolerance.
* **Fast Startup.** On Windows, shutting down with Fast Startup may keep the kernel running, so the next power-on wouldn't be a new `boot`.
  That needs testing, and may mean recording power cycles in their own right (see "Shutdowns").
* **Factory resets.** What a Windows "Reset this PC", or a phone's factory reset, changes: a new volume serial for `filesystem/root`, a
  new `kernel`, a lost device key. Whether it should be recorded as one event.
* **Driver stacks and release numbers.** Which part a graphics `version` names where there are several, such as a Linux kernel module and
  Mesa, and whether to record a maker's own release number such as `560.94`.
* **Settings that move scores.** Hardware-accelerated GPU scheduling, virtualisation-based security, Resizable BAR, memory profiles (XMP
  and EXPO), the power plan and Game Mode all change benchmark results. The owner does them to the device, so they look like events, and
  they need a kind of item.
* **Hybrid and external graphics.** Laptops with two adapters, and external ones that come and go.
* **Displays and peripherals.** Refresh rate and resolution matter to gamers, but monitors and controllers come and go.
* **Frame-rate definitions.** "1% lows" are calculated in several ways by different tools. Whether to match the most common one, or record
  percentiles of frame time instead.
* **Traces.** Whether a frame trace of a benchmark run may be kept as evidence, with what size limit, and where a host should keep it.
* **Naming titles.** A store and ID scheme for games, and how much of it is personal data.
* **Privileges.** Which sources need more than a standard service account or user.

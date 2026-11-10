## Device vocabulary (`tig-device/1`)

This vocabulary describes computing devices, such as desktops, servers and laptops, as recorded by their own agent (`tig-agent`). It
adds the meaning that `EVENTS.md` leaves to vocabularies: entity attributes, item kinds, event and action types, and stream names. It is a
draft: [Open questions](#open-questions) lists what is still undecided.

The vocabulary is platform-neutral. [Linux sources](#linux-sources) says where a Linux agent finds each value; agents for other
platforms map their own.

### What is recorded

> Record what describes the device; leave out what describes its owner. Ask: would a future owner mind seeing it?

A device's repository goes with the device when it is sold or handed on, so it is written for the next owner as much as the current one.

| Recorded | Not recorded |
| --- | --- |
| Hardware identity: model and serial numbers, MAC addresses | IP addresses, network names (e.g. Wi-Fi SSIDs), VPN configuration |
| Components: firmware, CPUs, memory, disks, network adapters, batteries | Hostname, which the owner chooses and the next owner will change |
| Filesystems on internal disks, the operating system and kernel | User accounts, home directories, files |
| Boots | Network shares, removable media, containers and what runs on the device |

Hardware identity is the device's equivalent of a car's VIN. Repositories are not generally public, but they may be shown to a buyer.

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
| `os` | `os` | `id`, `version`, `name`, `kernel` |
| `battery` | `battery/<hash>` or `battery/<name>` | `manufacturer`, `model`, `serial`, `technology`, `design-energy`, `design-charge` |

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

Filesystems of a persistent type (e.g. `ext4`, `xfs`, `btrfs`, `f2fs`, `zfs`, and `vfat` for an EFI system partition) on recorded disks.
Pseudo, temporary, image and network filesystems (`proc`, `tmpfs`, `overlay`, `squashfs`, `nfs`, `cifs`, ...) are not recorded.

* The ID is the mount point, which is a role: `/` is `filesystem/root`, and any other mount point is its path without the leading `/`,
  lowercased, with any character an item ID doesn't allow replaced by `-` (so `/boot/efi` is `filesystem/boot/efi`).
* A filesystem mounted at several points (e.g. by a bind mount) is recorded once, at its shortest mount point. Subvolumes mounted
  separately, such as btrfs `@` at `/` and `@home` at `/home`, are separate filesystems.
* `size` is the filesystem's total size. `disks` lists the IDs of the disks it is stored on, through any partitions, device-mapper
  (e.g. LVM, LUKS) or RAID layers.
* A filesystem recreated at the same mount point (a new `uuid`) is `replaced`: a `remove` and an `add` of the same ID.

#### Network adapters

Physical network adapters only: virtual interfaces (bridges, VPN tunnels, container and loopback interfaces) are not recorded. `mac` is
the permanent address burnt into the adapter, not a randomised or assigned one, so privacy features such as MAC randomisation don't look
like a new adapter. An adapter that has no permanent address is not recorded. `medium` is `ethernet` or `wifi`, `driver` is the driver's
name, and `hardware` is the adapter's ID in its bus's scheme, e.g. `{ system: pci, value: "8086:15f3" }`.

#### Operating system

`id` and `version` identify the distribution and its release (e.g. `ubuntu`, `"24.04"`), and `name` is its full name for people
(e.g. `Ubuntu 24.04.1 LTS`). A change to any of them is `upgraded`, including point releases. `kernel` is the running kernel's release,
so it changes only at a boot (see [Boot](#boot)).

#### Batteries

Batteries that power the device. `design-energy` or `design-charge`, whichever the battery reports, is its capacity when new. Its
current capacity is telemetry (see `TELEMETRY.md`, "Telemetry or state?").

### Events

| Event type | Recorded | Actions |
| --- | --- | --- |
| `discovery` | When the agent first runs, and when a new agent version first records a kind of item | `discovered` |
| `boot` | Once per boot | `booted` |
| *(none)* | When the agent finds that a component or the operating system has changed | `installed`, `removed`, `replaced`, `resized`, `upgraded`, `reconfigured` |

| Action type | Means | Changes |
| --- | --- | --- |
| `discovered` | Items found already present | `add`, each with `since` |
| `booted` | The device started | `update` of `os` `kernel`, only if it differs |
| `installed` | A component added | `add` |
| `removed` | A component taken out | `remove` |
| `replaced` | A component swapped for another, or a filesystem recreated | `remove` then `add`, and `update` of items that referred to the old one |
| `resized` | A capacity or size changed, e.g. memory added or a filesystem grown | `update` |
| `upgraded` | Firmware or the operating system updated | `update` |
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
| `boot` | The boot's unique ID, from the operating system | Text, e.g. a UUID |

The agent records each boot once, when it starts and finds that the boot's ID has no event yet. The boot in progress when the agent first
runs happened before the discovery event, so its `booted` action has no changes.

#### Changes

The agent finds changes by comparing the device with state replayed to the current time. It does so when it starts and periodically
while it runs (e.g. at each commit). Each difference is one action, of the type in the table above; a component removed and another of the
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
| `memory` | `available` | Bytes |
| `filesystem.<mount>` | `used`, `available` | Bytes |

`<mount>` is the filesystem's ID after `filesystem/`, with `/` replaced by `.`, e.g. `filesystem.root` or `filesystem.boot.efi`.

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

### Linux sources

Where a Linux agent finds each value. All are readable by an unprivileged user except the device's serial number, which an agent that
runs unprivileged, as `tig-agent` does, reads once with privileges when it is installed (e.g. in the package's install script).

| Value | Source |
| --- | --- |
| `entity` `vendor`, `model`, `serial` | `/sys/class/dmi/id/sys_vendor`, `product_name`, `product_serial` (root only) |
| `entity` `architecture` | `uname()` machine |
| `entity` `virtualisation` | `systemd-detect-virt` |
| `firmware` | `/sys/class/dmi/id/bios_vendor`, `bios_version`, `bios_date` (`MM/DD/YYYY`) |
| `cpu` | `/proc/cpuinfo`: `physical id`, `model name`, `cpu cores`, `siblings` |
| `memory` `capacity` | `/sys/devices/system/memory/block_size_bytes` × the number of `memory*/online` that are `1`; else `MemTotal` in `/proc/meminfo`, rounded up to a GiB |
| `disk` | `/sys/block/<dev>`: `size` × 512, `removable`, `queue/rotational`; `ID_MODEL`, `ID_SERIAL_SHORT`, `ID_WWN`, `ID_BUS` and `ID_PATH` in `/run/udev/data/b<major>:<minor>` |
| `filesystem` | `/proc/self/mountinfo` (mount point, root, device, type); `statvfs()`; `/dev/disk/by-uuid`; disks through `/sys/dev/block/<major>:<minor>/slaves` |
| `network` | Interfaces in `/sys/class/net` that have a `device`; permanent MAC from `ETHTOOL_GPERMADDR`, or `address` where `addr_assign_type` is `0`; `wireless` or `phy80211` for Wi-Fi; `device/vendor` and `device/device`; `device/driver` |
| `os` | `ID`, `VERSION_ID` and `PRETTY_NAME` in `/etc/os-release`; `uname()` release |
| `battery` | `/sys/class/power_supply/<name>` where `type` is `Battery`: `manufacturer`, `model_name`, `serial_number`, `technology`, `energy_full_design` (µWh) or `charge_full_design` (µAh) |
| Boot `time`, `boot` | `btime` in `/proc/stat`; `/proc/sys/kernel/random/boot_id` |

### Open questions

* **Other writers.** Version 1 assumes the device's agent is its only writer, as `tig-agent` does. An owner adding a repair, with the
  shop's invoice as evidence, would be valuable. Should the vocabulary allow it, and which events may other signers record?
* **Packages.** Installed software is done to the device, but a Linux system has thousands of packages that update almost daily. If
  recorded, should it be one event per package-manager transaction, and only for packages the owner chose?
* **Shutdowns.** Whether a boot should record how the previous one ended (cleanly, or by a crash or power loss), as part of the device's
  reliability history.
* **More components.** Memory modules, graphics cards, displays and peripherals. Some need privileges to identify.
* **Per-action `not_before`.** A change found at a boot is a separate event so that the boot's time stays exact. Should `EVENTS.md` let an
  action narrow the event's time instead?
* **Reinstalls.** A reinstalled operating system loses the device key, so it starts a new repository (see `ROADMAP.md`, "Device key
  lifecycle"). Should the new repository refer to the old one?

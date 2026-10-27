## Telemetry

Telemetry is point-in-time data that does not change the state of an entity, but may be analysed for trends (see the whitepaper).
This document describes how telemetry is laid out, written and verified. It is a draft: the API shown is indicative, and
[Open questions](#open-questions) lists what is still undecided.

### Telemetry or state?

> Telemetry is *observed*; events are *done to* the entity.

A value that is measured, even one that changes slowly, is telemetry. A change made to the entity is an event, which updates state.

| Value | Kind | Why |
| --- | --- | --- |
| Free disk space | Telemetry | Observed |
| Battery health | Telemetry | Observed, even though it wears down slowly |
| Battery replaced | Event | Done to the device |
| Disk capacity | State | Changed only by an event, such as a disk being replaced |

An agent may *detect* a change from telemetry (e.g. a different disk capacity) and then record the matching event.

### Streams

Telemetry is grouped into **streams**. A stream is one measured quantity from one source, such as `cpu.temp`, `disk.root.used` or `odometer`.

* Stream names are lowercase letters, digits, `-` and `.`, starting with a letter or digit. Dots separate namespaces.
* Each stream has its own folder under `/telemetry`, named after the stream.
* A stream is described by a `stream.yaml` file in its folder, so the data is self-describing.

```
telemetry/
  cpu.temp/
    stream.yaml
    2026/
      2026-10-22.csv     # sealed
      2026-10-23.csv     # latest chunk, still being appended to
  odometer/
    stream.yaml
    2026.csv
```

#### Stream definition

```yaml
# telemetry/cpu.temp/stream.yaml
description: CPU package temperature
unit: celsius
columns:
  - name: value
    type: decimal
chunk: day
max_lag: 5m
```

| Field | Description | Format |
| --- | --- | --- |
| `description` | What the stream measures (optional) | Text |
| `unit` | The unit of the value columns | Text, e.g. `celsius`, `bytes`, `miles` |
| `columns` | The value columns, after `time` | List of `name` and `type` (`integer`, `decimal` or `text`) |
| `chunk` | How readings are grouped into files | `hour`, `day`, `month` or `year` |
| `max_lag` | How long after sampling a reading may still be written | Duration, e.g. `30s`, `5m`, `1h` |

A stream definition is written once, in the same commit as the stream's first reading, and never changed afterwards. Changing a unit or
a column would change the meaning of readings already recorded, so a stream that needs a different shape is a new stream.

### Chunks

A stream's readings are stored in **chunk** files, each covering one period in UTC. The period is chosen to suit how often the stream is sampled:

| `chunk` | Path | Suits |
| --- | --- | --- |
| `hour` | `YYYY/MM/YYYY-MM-DDTHH.csv` | Several readings a second |
| `day` | `YYYY/YYYY-MM-DD.csv` | A reading every few seconds to every few minutes |
| `month` | `YYYY/YYYY-MM.csv` | A reading every hour or so |
| `year` | `YYYY.csv` | Occasional readings, e.g. an odometer read at each service |

A chunk is **sealed** once the period it covers has ended and its `max_lag` has passed. Readings are only accepted in time order and within
`max_lag` (see [Writing](#writing)), so a sealed chunk can never receive another reading, and each chunk holds exactly the readings for its period.

#### Append-only rule

Within `/telemetry`, a commit may only:

* add a new stream (its `stream.yaml` and first chunk),
* append rows to the end of a stream's latest chunk, or
* add a stream's next chunk.

Any other change, such as editing, reordering, deleting or renaming rows or files, breaks the rule. Verifiers can check this mechanically
across a repository's history.

History is never rewritten, for example to downsample old readings, because that would invalidate signatures. A stream's resolution is
decided when it is written. Aggregates, such as hourly averages, belong in `/summaries`.

### File format

Chunks are [CSV](https://www.rfc-editor.org/rfc/rfc4180) files in UTF-8 with LF line endings. The first row is a header: `time`, then the
stream's columns in the order they are defined.

```
time,value
2026-10-23T10:15:00Z,47.5
2026-10-23T10:16:00Z,48.0
```

* `time` is the time the reading was sampled, in [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) UTC with a `Z` suffix, with optional
  fractional seconds to millisecond precision.
* Times strictly increase within a stream.
* Decimals use `.` and no thousands separators. An empty field means no value.

CSV keeps uniform readings compact, diffs line by line and opens in any spreadsheet. As a guide, a reading every minute is about 26 bytes,
or about 37 KB a day and 13 MB a year before Git compresses it.

### Writing

Readings are written to the repository as soon as they are sampled. When a reading is appended, the writer rejects it unless:

* its time is no earlier than *now* − `max_lag`,
* its time is no later than *now*, allowing a small skew, and
* its time is later than the stream's previous reading.

*Now* is the TiG clock (see below), not the device's clock alone.

Importing historical data, or writing data that was buffered for longer than `max_lag`, is out of scope for this draft. It may later be
allowed through a separate API that marks the data as such, so that verifiers can treat it differently.

#### Cadences

Sampling, committing and signing happen at different rates:

| Cadence | Example | Set by |
| --- | --- | --- |
| Sampling | Every 60 seconds | The agent, per stream |
| Committing | Every 15 to 60 minutes, and when a chunk is sealed | The agent; a trade-off between data lost in a crash and the number of commits |
| Signing | Hourly or daily | The agent; one signature covers all earlier history (see `COMMITS.md`) |

Readings are written straight away but committed in batches, so one commit usually holds many readings.

### TiG clock

The TiG clock is the device's clock, corrected by an offset measured against TiG Notary. It exists so that an honest device whose clock has
drifted does not have its readings rejected.

* **Baseline.** The operating system's clock, normally kept in step by NTP.
* **Offset.** Every signing request is a round trip to the notary, and the returned tag's `tagger` time is the notary's clock. The offset is
  estimated as the tag time minus the midpoint of the request's send and receive times. Tag times have one-second resolution, which is
  enough for checking `max_lag`.
* **Offline.** The last known offset is used, with the device's monotonic clock measuring time since it was taken. The offset is stored
  locally on the device and never committed.
* **Monotonic.** The clock never goes backwards. If the operating system's clock jumps back, it holds its last value until it catches up.

### Verifying

Notary signatures prove only that a commit existed *no later than* the signature's time (see `COMMITS.md`). For telemetry, a verifier can check:

1. **Append-only.** Every commit follows the [append-only rule](#append-only-rule).
2. **Not future-dated.** Every reading added in a commit has a time no later than the earliest signature covering that commit, allowing a small skew.
3. **Delay.** The gap between a reading's time and the earliest signature covering it. A gap much longer than `max_lag` is not a violation,
   since devices may be offline, but verifiers may flag it.

Signatures do not prove that a reading was *not backdated*, because nothing shows that a commit was made *after* a given time.
See [Open questions](#open-questions).

### Library API

`tig-lib` implements this draft (see `include/tig/tig.h`):

```c
tig_status tig_clock_now(tig_repository* repository, int64_t* unix_ms);

tig_status tig_telemetry_stream_define(tig_repository* repository, const char* stream,
                                       const tig_stream_definition* definition);
tig_status tig_telemetry_append(tig_repository* repository, const char* stream, int64_t unix_ms,
                                const char* const* values, size_t value_count);

tig_status tig_repository_commit(tig_repository* repository, const char* message);
```

* `tig_telemetry_stream_define` writes `stream.yaml`, and succeeds without changing anything if the stream is already defined identically.
* `tig_telemetry_append` applies the checks in [Writing](#writing) and owns chunk paths and the CSV format, so callers never handle
  telemetry files themselves. Each reading is written to the working tree straight away, so it survives a crash before the next commit.
* `tig_repository_commit` commits everything written since the last commit, after checking the [append-only rule](#append-only-rule).
* A test clock can be supplied through `tig_options`.

Not yet implemented: correcting the TiG clock by the notary offset, and signing commits with the device key.

### Open questions

* **Proving readings weren't backdated.** A commit could carry a trailer quoting the latest signature tag, e.g. `TiG-After: <tag hash>`, which
  would show the commit was made after that time and give readings a lower bound.
* **Importing and buffered data.** The separate API mentioned in [Writing](#writing), and how verifiers treat what it writes.
* **Several values per sample.** When should related values, such as per-core CPU temperatures, be columns of one stream rather than separate streams?
* **Well-known stream names.** Whether each vertical needs a registry of names (e.g. `odometer` for GTdb), so that tools agree on them.
* **Recording clock offset.** Whether the TiG clock's offset should itself be recorded as a stream, so verifiers can see how far a device's clock drifted.
* **Allowed skew.** The value of the small skew allowed for readings and signatures.

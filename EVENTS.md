## Events and state

An event is something done to an entity at a point in time: a car's oil changed, a boiler installed, a medicine prescribed, a disk
replaced. Events are the only way that state changes. This document describes how state and events are laid out, written, replayed and
verified. It is a draft: [Open questions](#open-questions) lists what is still undecided.

### Events, state or telemetry?

> Telemetry is *observed*; events are *done to* the entity (see `TELEMETRY.md`).

|  | Vehicular | Residential | Medical | Device |
| --- | --- | --- | --- | --- |
| State | VIN, tyres, engine oil, MOT certificate | Rooms, boiler, appliances | Date of birth, conditions, allergies, medications | Disks, memory, OS |
| Event | A service that replaces the oil and filter | A boiler replaced | A diagnosis; a prescription started or stopped | A disk replaced; a boot |
| Telemetry | Odometer | Solar generation | Blood cholesterol | Free disk space |

Values derived from state and telemetry, such as the miles a component has done since it was fitted (from the odometer reading in the
event that fitted it) or a person's age, are not stored.
They are calculated when needed, and may be cached in `/summaries`.

### Model

**Items.** State is a set of *items*, each with an ID and attributes. An item is anything with a history of its own: a tyre, the engine
oil, a boiler, a room, a condition, a medication, a disk. The entity itself is the item `entity`, which holds attributes of the whole, such
as a car's VIN or a person's date of birth.

**Events and actions.** An event is one occasion, such as a service, a contractor's visit, an appointment or a boot. It has a time and
lists *actions*, each of which is one thing done, such as replacing the oil or prescribing a medicine.

**Changes.** An action changes state through a list of *changes*, which use only three operations: `add`, `update` and `remove`. These
are the same in every domain, so any tool can rebuild state by replaying events without knowing what a service or a prescription is. What
an action means is given by its type, from a [vocabulary](#vocabularies).

An action may have no changes when nothing about state changed, e.g. an inspection that found nothing wrong. Something that is only
observed, with nothing done, is telemetry rather than an event.

### Repository layout

```
state/
  initial.yaml                 # optional
events/
  1998/
    1998.p0x4rt.yaml           # time known only to the year
  2024/
    2024-03-11.7fd2ma.yaml
  2026/
    2026-11-04.q9w3ne.yaml     # an instant, named by its UTC date
```

* Each event is one file, at `events/YYYY/<time>.<id>.yaml`. `<time>` is the event's [time](#times) to the precision it's given, but
  no finer than a day, so `1998`, `1998-06` or `1998-06-14`. Events timed to the instant use their UTC date.
* An event ID is 6 to 16 lowercase letters and digits, chosen at random, and unique within the repository. Choosing at random lets
  several writers, such as a device and its owner's app, add events without coordinating.

### Initial state

`state/initial.yaml` is the state that events are replayed onto. It holds what was known about the entity before its first recorded
event and needs no time of its own: a car's VIN and build date, a house's construction, a person's date of birth.

```yaml
# state/initial.yaml
kind: vehicle
vocabulary: gtdb/1
items:
  entity:
    vin: "SAJAA0000000X0000"
    make: Example
    model: Roadster
    built: "2009-05"
```

| Field | Description | Format |
| --- | --- | --- |
| `kind` | What the entity is | `vehicle`, `residence`, `person`, `device`, ... |
| `vocabulary` | The vocabulary its events use (optional) | Name and version, e.g. `gtdb/1` |
| `items` | The items, keyed by ID (optional) | Map of item ID to attributes |

* The file may be corrected like an event (see [Corrections](#corrections)). A repository without one starts from empty state.
* It is a starting point, not a snapshot at a particular time: replay applies it before every event, including events dated before the
  file was written.
* Facts learned later go in events, not in a correction to this file: a correction says the file was wrong from the start, which is a
  different claim. A writer that finds items already present, such as a used car's components or a computer's disks, adds them in an
  event with `since` (see [Changes](#changes)).

### Event format

Events are YAML files: one document, UTF-8, LF line endings. Times are strings and should be quoted, because some YAML parsers otherwise
turn them into dates.

```yaml
# events/2024/2024-03-11.7fd2ma.yaml
time: "2024-03-11"
type: service
by:
  name: Example Motors
  reference: "71236"
readings:
  odometer: 65807
evidence: [/evidence/2024-03-11-invoice.pdf]
actions:
  - type: inspected
    description: MOT test
    result: pass with advisories
    cost: { amount: 50.00, currency: GBP }
    evidence: [/evidence/2024-03-11-mot.jpg]
    changes:
      - update: certificate/mot
        set: { number: "1234 5678 9012", expires: "2025-03-22" }
  - type: replaced
    description: Engine oil, 5W-30
    cost: { amount: 62.90, currency: GBP, tax: 12.58 }
    changes:
      - remove: fluid/engine-oil
      - add: fluid/engine-oil
        set: { grade: 5W-30, quantity: { value: 7.4, unit: litres } }
```

| Field | Description | Format |
| --- | --- | --- |
| `time` | When it happened (required) | A [time](#times) |
| `not_before` | The earliest it can have happened, when it is only known to lie between two times | A [time](#times), no later than `time` |
| `type` | What kind of occasion it was (optional), so that questions like "when was my car last serviced?" have an answer | A type from the vocabulary, e.g. `service`, `appointment` |
| `by` | Who did it, such as a garage, contractor, clinic or the owner | Map; `name`, `role`, `address` and `reference` are suggested |
| `source` | How the event was recorded, if that isn't obvious | Text, e.g. `tig-agent 0.5.0` or `entered from a paper invoice` |
| `readings` | What was observed during the event | Map of stream name to value (see below) |
| `evidence` | Files that support the whole event | List of paths under `/evidence` |
| `cost` | The total cost, when it isn't given per action | [Cost](#costs) |
| `notes` | Anything else | Text |
| `actions` | What was done (required) | List of actions |

| Action field | Description | Format |
| --- | --- | --- |
| `type` | What was done (required) | A type from the vocabulary, e.g. `replaced`, `prescribed`, `booted` |
| `description` | What was done, for people | Text |
| `changes` | How state changed | List of [changes](#changes) |
| `cost`, `evidence`, `notes` | As for the event, but for this action only | |

Vocabularies may add fields to events and actions, such as an MOT test's `result`. Readers must keep fields they don't understand and
otherwise ignore them.

**Readings** are telemetry observed during an event, such as the odometer at a service or blood pressure at an appointment. They don't
change state. They are kept in the event, rather than appended to a stream, because events are often recorded long after they happened and
a stream only accepts recent readings (see `TELEMETRY.md`, "Writing"). Keys are stream names, and values are a scalar or, for a stream
with several columns, a map of column to value.

#### Times

| Form | Example | Means |
| --- | --- | --- |
| `YYYY` | `"1998"` | Some time in that year |
| `YYYY-MM` | `"1998-06"` | Some time in that month |
| `YYYY-MM-DD` | `"2024-03-11"` | That calendar day, where it happened, as on a receipt |
| [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) UTC with `Z` | `"2026-11-04T07:12:31Z"` | That instant, with optional milliseconds |

Use the most precise form known. Dates have no timezone; instants are UTC, as in telemetry.

Unlike telemetry, events may be written long after they happened: entering a 2018 invoice in 2026 is normal. They must not be dated in
the future.

`not_before` covers changes that are noticed after the fact. A disk swap found when a computer boots happened at some point after the
last reading before it shut down, and a fault found at a service arose at some point after the previous one.

#### Costs

| Field | Description | Format |
| --- | --- | --- |
| `amount` | The cost before tax | Decimal |
| `currency` | The currency | [ISO 4217](https://www.iso.org/iso-4217-currency-codes.html) code, e.g. `GBP` |
| `tax` | The tax on top of `amount` (optional) | Decimal |

### Changes

```yaml
changes:
  - add: tyre/a4519            # a new item; the ID must not exist
    set: { position: front-left, size: 225/45 R17 }
  - update: tyre/b4519         # an existing item
    set: { position: rear-left }
    unset: [advisory]
  - remove: tyre/c4519         # an existing item
```

| Field | Applies to | Description |
| --- | --- | --- |
| `add`, `update` or `remove` | | The operation, with the item's ID as its value. Each change has exactly one. |
| `set` | `add`, `update` | Attributes to set. Each replaces the whole of that attribute's value. |
| `unset` | `update` | Attributes to delete |
| `since` | `add` | When the item really became part of the entity, if that was before the event: a [time](#times), or `unknown` |

* Item IDs are lowercase letters, digits, `-`, `.` and `/`, starting with a letter or digit. The part before the first `/` is the item's
  kind from the vocabulary, such as `tyre`, `condition` or `disk`. An ID without a `/` is an item there is only ever one of, such as
  `memory`. `entity` is reserved for the entity itself.
* An ID names one thing for as long as it is part of the entity. A tyre moved to another wheel is an `update` of its position, not a
  `remove` and an `add`. Where individual things aren't worth telling apart, such as engine oil or a filter, the ID can name the role, and
  replacing it is a `remove` and an `add` of the same ID.
* Changes apply in order, so an action may remove an ID and then add it again.
* `since` separates "fitted at this time" from "was already there, and first recorded now". Replay still adds the item at the event's
  time; `since` is information for readers. If the start is itself a known event, such as an installation with a receipt, record that
  event at its own time instead.
* Removing an item ends it, but its history stays in the events. Where it went (sold, scrapped, moved to another entity) is up to the
  action type.

Attribute values may be any YAML scalar, list or map. The vocabulary gives each attribute's meaning and unit. A quantity that has no unit
in the vocabulary is written `{ value, unit }`. A value taken from an external scheme, such as a part number or a clinical term, is
written `{ system, value }`, e.g. `{ system: snomed-ct, value: "38341003" }`.

Attributes may hold dates in the future, such as an expiry, but events only ever record what has happened.

### Replay

State at any time is rebuilt by replaying events onto the initial state:

1. Start from `state/initial.yaml`, or empty state if there isn't one.
2. Order events by time: by the start of the period each `time` covers, then coarser before finer (`1998` before `1998-06-14`), then by
   ID. Dates are compared as if they were UTC.
3. Apply each action's changes in order. `add` fails if the ID exists; `update` and `remove` fail if it doesn't.

State at time *T* is the result of replaying the events whose time starts no later than *T*. The result has the same shape as
`initial.yaml`, and may be cached in `/summaries` but is never committed.

Replay must succeed. A writer must reject an event that would make any step fail, including for events dated after it, since a backdated
event can conflict with history that is already recorded.

### Corrections

A mistake is corrected by editing the event's file, or deleting it, in a new commit. The earlier version stays in Git history, signed and
timestamped like any other commit, so a correction is just as visible to a verifier as the mistake was.

```diff
 # events/2024/2024-03-11.7fd2ma.yaml
       - add: fluid/engine-oil
-        set: { grade: 5W-30, quantity: { value: 7.4, unit: litres } }
+        set: { grade: 5W-30, quantity: { value: 4.7, unit: litres } }
```

* Correct a record when it was wrong; add an event when the entity changed. Fixing a mistyped oil quantity is a correction, but topping
  up the oil is a new event.
* An event's ID is its identity. If a correction changes the event's time, the file moves to the matching path and keeps its ID. An ID is
  never reused, even after its event is deleted, so the history of each ID is unambiguous.
* A correction is always a new commit. Git history is never rewritten to make one (e.g. with `git commit --amend`, or by rebasing
  commits that are already signed), since that would invalidate signatures.
* The reason can be given in the event's `notes` or in the commit message.

Because corrections are commits, Git gives two views of the timeline. Replaying the latest commit gives the best-known history, and
replaying an earlier commit gives history as it was recorded then, e.g. what a buyer was shown before a sale.

### Writing

A commit may add, correct, move or delete event files, and add, correct or delete `state/initial.yaml`. When an event is added or
corrected, the writer rejects it unless:

* it is valid YAML in the format above, and its path matches its time and ID,
* a new event's ID has never been used in the repository, and a corrected event keeps its ID,
* every evidence file it lists is in the same commit or an earlier one,
* its time is no later than *now*, by the TiG clock (see `TELEMETRY.md`), and
* replay still succeeds.

Deleting an event, or correcting the initial state, is likewise rejected if replay would then fail.

A repository may have several writers, such as a car owner's app and a garage, each signing its own commits (see `COMMITS.md`,
"Several signers"). Their histories are combined with merge commits, and a merge must leave replay succeeding too. A conflict, such as
both writers adding the same item, is resolved in the merge like any other Git conflict, by correcting or deleting one side's event.

`by` is a claim: a signature shows which key recorded an event, not that the named garage did the work.

### Verifying

A verifier can check:

1. **Valid.** Every event is well formed, and replay succeeds at every commit.
2. **Not future-dated.** Every version of every event has a time no later than the earliest signature covering that version, allowing a
   day for dates, which have no timezone.
3. **Delay.** The gap between an event's time and the earliest signature covering its first version. Backfilled events are normal, but
   verifiers may show them differently from events recorded as they happened, e.g. "recorded 6 years later".
4. **Corrections.** Every commit that corrected or deleted an event, or the initial state: which key made it, and how long after the
   earliest signature covering the original. Corrections are normal, but verifiers may flag those made long afterwards, or by a different
   signer from the one that wrote the event.

### Vocabularies

The rules above are the same for every kind of entity. A vocabulary adds the meaning for one domain: entity attributes, item kinds and
their attributes and units, event and action types and their extra fields, and the stream names used in `readings`. It may also require
a single signer, as a device vocabulary would for repositories written only by the device's own agent. Vocabularies will be specified
separately; this is a sketch of what each might cover.

|  | Device | Vehicle (GTdb) | Residence | Person |
| --- | --- | --- | --- | --- |
| Entity attributes | Model | VIN, make, model, build date | Construction date, type | Date of birth, gender |
| Item kinds | `disk`, `filesystem`, `memory`, `os`, `package` | `tyre`, `fluid`, `filter`, `battery`, `certificate` | `room`, `boiler`, `appliance`, `window`, `certificate` | `condition`, `allergy`, `medication`, `immunisation`, `implant` |
| Event types | | `service`, `mot`, `purchase` | `visit`, `survey` | `appointment`, `admission` |
| Action types | `discovered`, `installed`, `removed`, `resized`, `upgraded`, `booted` | `acquired`, `fitted`, `replaced`, `repaired`, `inspected` | `built`, `installed`, `replaced`, `repaired`, `inspected` | `recorded`, `diagnosed`, `resolved`, `prescribed`, `stopped`, `immunised`, `operated` |
| Readings | | `odometer` | Meter readings | Weight, blood pressure |
| External schemes | | Part numbers | | SNOMED CT, dm+d |

#### Examples

A residence: a boiler replaced.

```yaml
# events/2025/2025-09-02.h2b8xq.yaml
time: "2025-09-02"
by: { name: Example Heating }
evidence: [/evidence/2025-09-02-boiler-invoice.pdf]
actions:
  - type: replaced
    description: Gas combi boiler
    cost: { amount: 2000.00, currency: GBP, tax: 400.00 }
    changes:
      - remove: boiler
      - add: boiler
        set: { make: Example, model: Combi 30, fuel: gas, location: kitchen, warranty-expires: "2035-09-02" }
```

A person: a GP appointment that diagnoses a condition, prescribes a medicine, and records a long-standing allergy.

```yaml
# events/2026/2026-02-14.m4c7zp.yaml
time: "2026-02-14"
type: appointment
by: { role: GP }
readings:
  blood.pressure: { systolic: 152, diastolic: 96 }
actions:
  - type: diagnosed
    changes:
      - add: condition/hypertension
        set: { name: Hypertension, code: { system: snomed-ct, value: "38341003" } }
  - type: prescribed
    changes:
      - add: medication/amlodipine
        set: { name: Amlodipine, dose: { value: 5, unit: mg }, frequency: daily }
  - type: recorded
    changes:
      - add: allergy/penicillin
        since: "1998"
        set: { name: Penicillin }
```

A device: a disk swap found at boot, recorded by its agent.

```yaml
# events/2026/2026-11-04.q9w3ne.yaml
time: "2026-11-04T07:12:31Z"
not_before: "2026-11-03T22:41:00Z"
source: tig-agent 0.5.0
actions:
  - type: replaced
    changes:
      - remove: disk/3f9a1c
      - add: disk/b71e05
        set: { model: Example NVMe 2TB, capacity: 2000398934016 }
  - type: booted
    boot: 8c2e4f1a-6b0d-4c55-9e3a-2f1d7c9b0a64
    changes:
      - update: os
        set: { kernel: 6.8.0-48-generic }
```

### Library API

Not yet implemented. `tig-lib` will need to write and correct `state/initial.yaml`; add, correct and delete events, choosing new IDs and
paths and applying the checks in [Writing](#writing); replay state to a given time; and stage `/events` and `/state` in
`tig_repository_commit`, which currently stages only `/telemetry`.

### Open questions

* **Signer roles.** Any listed signer can add another and write any event. Should a vocabulary limit who records what, e.g. only a
  clinician's key records a diagnosis? May one signer correct or delete an event that another wrote, such as an owner editing a garage's
  record? And how is a key tied to the garage or clinic named in `by`?
* **Items moving between entities.** An engine taken from a donor car, an appliance moved to a new house or a transplanted organ: how the
  `remove` in one repository is linked to the `add` in another.
* **Ownership.** Buying and selling are events, but owners' names are personal data. Should ownership be recorded in events at all, or only
  through transfer (see `ROADMAP.md`, phase 3)?
* **Privacy.** Medical events are special category data, and `by` can name individuals, such as a GP or a mechanic. Encrypting subsets of
  history (`ROADMAP.md`, phase 3) may need to work per event, or even per action.
* **Vocabulary registry.** Where vocabularies are published, how they are versioned, and whether one entity may use several (e.g. a
  house's boiler described with device terms).
* **Real time.** Replay adds an item at the time of the event that adds it, and Git history already shows what was recorded when. Should
  `since` also let replay show state as it really was, e.g. with an allergy present from 1998 rather than from the appointment that
  recorded it?
* **Dates and time zones.** Comparing dates as UTC can misorder a date-only event and an instant near midnight.
* **Where `kind` lives** in a repository without `state/initial.yaml`; perhaps `/.tig`.
* **Planned events.** Appointments, recalls and services that are due haven't happened, so they aren't events. Do they belong in a
  repository at all?
* **Evidence.** Whether evidence files can be corrected like events, or must never change once an event refers to them. A future
  evidence spec should decide.

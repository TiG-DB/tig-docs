## Evidence

Evidence is the unstructured files that support events: a photo of an MOT certificate, a scanned invoice, a boiler's warranty
card. They are for people to check, and nothing reads data from them (see `WHITEPAPER.md`). This document describes how they are named,
added and kept. It is a draft: [Open questions](#open-questions) lists what is still undecided.

### Layout

```
evidence/
  2024-03-11-invoice.jpg
  2024-03-11-mot-certificate.jpg
```

* Evidence files live in `/evidence`. An event, or one of its actions, lists the files that support it in `evidence`, by path (see
  `EVENTS.md`). One file may support several events, such as a service history that lists every visit.
* A file's name is 1 to 100 characters: letters, digits, `.`, `_` and `-`, starting with a letter or digit. Names should say what the
  file is, usually starting with its date, so that a folder listing reads like the timeline: `2024-03-11-invoice.jpg`. Phone and
  scanner file names such as `IMG_0412.JPG` are valid but say little.
* Writers put files directly in `/evidence`. Readers must accept any path under it, including folders, so that a repository written by
  another tool can still be read.

### Writing

A writer rejects a new evidence file unless:

* its name follows the rules above,
* no file of that name is already in the repository, so a path always means the same file, and
* it is not empty.

Evidence may be added in the same commit as the event that lists it, or in an earlier one (see `EVENTS.md`, "Writing"). Evidence that no
event lists yet is allowed: a receipt may be photographed before the event is entered.

### Evidence never changes

Once a commit contains an evidence file, no later commit changes or removes it. A writer rejects a commit that does. A file that turns
out to be the wrong one is fixed by adding the right one and correcting the events that list it, saying why in the event's `notes` or the
commit message.

This is stricter than for events, which may be corrected (see `EVENTS.md`, "Corrections"), because an event is a claim and its evidence
is what a reader checks that claim against. A reader of any past commit sees the evidence as it was then, and a signature over a
commit covers its evidence.

The same rule means a mistake cannot be removed. Git history keeps every version of every file, and rewriting history would invalidate
signatures (see `COMMITS.md`). A writer, and the person it acts for, should look at what a file shows before adding it, such as an
invoice with a card number, or a photograph with a location in its metadata.

### Verifying

A verifier can check, besides what `EVENTS.md` requires, that no commit changes or removes an evidence file that an earlier commit
contains.

### Open questions

* **Size.** Phone photos are several megabytes each, and Git keeps every file for ever. A limit, a rule for downsizing, or somewhere to keep
  large files outside Git (`ROADMAP.md`, "Hosted storage needs a business plan"), if one is wanted.
* **File types.** Whether the standard names the types a reader can expect (JPEG, PNG, PDF), or leaves it to the writer.
* **Metadata.** Whether a writer should strip location and device details from photographs when it adds them, or leave the file as it
  was taken.
* **Withdrawal.** Evidence may turn out to contain something that must not be kept, such as another person's data. Immutability and
  the right to erasure pull in opposite directions, and encrypting subsets of history (`ROADMAP.md`, phase 3) may be the way through.
* **Links by hash.** Whether an event should record a hash of each file it lists, so that a copy of the repository can show that a file was
  not swapped when it was moved without its history.
* **Several files for one document.** The pages of a V5C or the photos of one invoice are separate files today.

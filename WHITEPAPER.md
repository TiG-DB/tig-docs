# TiG

## Timeline In Git

# Executive Summary

TiG is an open standard in data storage that can be used to store timeline-based data. As it is intended to be able to store detailed historical data, potentially of a personal nature, it is designed to be both easy to isolate (secure, private) and yet accessible to inspection and transfer (verification, ownership).

We propose a structured file-based store using *Git* as both the storage medium and transport mechanism of data, with a defined schema for files and their contents.

This approach makes reasoning about and understanding the provenance of data simple, and will give confidence to users and other stakeholders.

# Introduction

Many user-centric views of the world require queries that need highly local, personal, historical data. For example, a person may ask “when did I last service my car?”, “who installed my boiler?” or “which vaccines did I have as a child?”. These three subjects (a car, a house and a medical history) will inform this white paper, but we envisage many more.

This data may need to change hands. For example, when a car or house is sold, or when a child becomes an adult and takes control of their health records.

This white paper proposes a solution that enables the storage, querying and transferring of this data in a robust manner with a model that is open, easy to reason about and long lasting.

Examples will be illustrated thus:

| Vehicular | Residential | Medical |
| :---- | :---- | :---- |
| A timeline of the maintenance service history of a vehicle. | A timeline of the development and maintenance of a residence. | A timeline of medical tests and procedures carried out on a person. |

# Problem Statement

The current (2024) IT world is dominated by platforms, which store data opaquely on behalf of users and grant access to said users and 3rd parties via APIs. Open protocols and file formats have been eschewed in favour of APIs controlled by platform holders, trading implementation choice for convenience and rapid evolution.

For long-running services that need data to be stored for many years, and transferred in a frictionless manner, relying on platform holders is naive and disempowering to users. We may compromise and allow (say) Apple or Google to be custodians of our photos, but for important, rich, historical data that may need to be transferred wholesale between these platforms, the need for defined protocols and schemas re-emerges.

To design for the widest possible scope, the data also needs non-software-first organisations to be able to effectively contribute with minimum friction to their business processes and get tangible value from doing so.

Finally, this data needs strong provenance and validation facilities if this is to be a trusted interchangeable format. 

# Solution

This data in question is temporal \- often event or telemetry based, meaning the history is just as, or more important as the current status. Your heart rate may be 90 bpm at the moment, but without contextual information this cannot be reasoned about. Ditto your car’s fuel consumption or your house’s KwH usage.

Therefore, the data for an entity (car, house, person) should be stored in a data store that is temporal and file oriented. The current standard for such repositories is Git, with a share of [over 90%](https://stackoverflow.blog/2023/01/09/beyond-git-the-other-version-control-systems-developers-use/) of software developers, who use it to store their code, along with historical changes.

Events and telemetry, along with summary and current status should be stored as simple files. There are no complicated relationships that need to be mapped. Events are usually self describing and a standalone source of truth.

With simple conventions on top of Git’s core functionality, data provenance can be recorded at the time of its recording, along with supporting evidence in the form of human readable files.

# Technical Details

The specification for TiG will comprise of:

* A specification for repository layout.  
* A specification for the contents of each file type in the repository.  
* A reference implementation that is able to store and query data in the repository, along with unit tests and sample data.   
* A specification for transferring ownership of repositories.

## Entities

The data recorded will be *state*, *events*, *telemetry* and *evidence*.

### State

State normally exists in only two places, at the creation of entities (the initial state) and the current (cached) state. Intermediate states can be recovered from replaying events. Checkpoints of state may be taken at any point (e.g. for caching purposes), but these are outputs of the system, and not stored in the repository.

| Vehicular | Residential | Medical |
| :---- | :---- | :---- |
| The components and the number of miles since their installation. | The state of the roof, windows and appliances within a home. | Gender, age and known conditions. |

### Events

Events are the updates to state that happened at a point in time.

| Vehicular | Residential | Medical |
| :---- | :---- | :---- |
| A servicing event that changes the oil and filters.. | The installation of a new fridge/freezer. | The prescription of a medicine.   |

### Telemetry

Telemetry is point-in-time data that does not affect the state of the system, but may be analysed for trends..

| Vehicular | Residential | Medical |
| :---- | :---- | :---- |
| The current odometer reading. | The amount of energy being generated by a solar array. | A blood cholesterol measurement. |

### Evidence

Evidence is unstructured files that support events. They are for human verification purposes and do not have any data within them. For example, 

| Vehicular | Residential | Medical |
| :---- | :---- | :---- |
| A picture of an MOT test certificate. | The receipt for an alarm installation. | A chest X-ray. |

###  

## Repository Layout

### Entities

Entities will be stored in the following folders  
/telemetry  
/events  
/evidence  
/state

### Orientation files

Every repository carries `README.md` and `AGENTS.md` in its root. `README.md` tells a human what TiG is and what each folder holds. `AGENTS.md` gives AI agents and scripts the working rules: add events rather than rewrite, never rewrite history, telemetry is append-only, leave evidence and `.tig/allowed_signers` alone, never store secrets.

These are written, and committed in the first commit, by `libtig` when it creates the repository. Their text is compiled into the library, so creating a repository never needs the network or a template repository to be fetched. Both files are generic: they name no entity. Owners may extend them, and the library never overwrites an existing copy.

A canonical copy of the generic text may be published for reference, but it is not how the files are distributed.

### Summaries

Application specific denormalized data will be stored in /summaries, which is listed in the .gitignore file, and not replicated to servers etc.

The summaries directory is for placing non-replicated summary data which, for convenience, is stored physically inside the repository file system directory. Example usage might be an SQLite database that has telemetry data points ingested into tables for aggregation purposes.

### Metadata

TiG's own metadata about the repository is stored in /.tig. Unlike /summaries it is committed and replicated, as it is needed to interpret and verify the repository. For example:

* /.tig/allowed_signers: the public keys of the devices that sign commits (see COMMITS.md).

Future versions of the specification may add further files here, such as the version of the TiG format the repository follows.

## Verification of Commits

Whenever entities in the repository change a git commit is performed. This implicitly timestamps the operation. The authenticity of this timestamp, and the identity of the committer can be strengthened with a simple cryptographic signature.

Each commit is signed by the device that made it, and a standard Git signed tag pointing at the commit, created by a signing authority (a notary), confirms when it existed. Because a commit's hash covers its history, one signature also covers every earlier commit. See COMMITS.md for the convention.

# Benefits and Advantages

## File Oriented

Each event and chunk of telemetry is modelled and stored as a standalone file, simplifying representation. Individual events etc. are simple to examine.

## Familiar Infrastructure

No matter the technology stack that a software developer uses, one of the most generally familiar and understood tools is a Git repository. It offers a layer of workflow and temporal infrastructure that naturally maps to the problem domain, and will be easily understood.

## Always Local

As data is always available locally (as a cloned repository) it is available without access to the Internet. This may be critical in some situations. 

## Simple Encapsulation

While it is more complicated than having a single document or a flat directory of files, it is understood well enough to easily say “this is the totality of data for this timespan”. A backend of (say) a conventional database would be much harder to reason about without specialist tools.

## Simple Validation

As events are time-stamped and signed, selective editing of data is non-trivial.  Likewise, synthetic data generation is also very difficult.

# Challenges and Considerations

## Unorthodox Approach

Using a Git repository for such data is a novel approach that may appear to some to be a risky choice and doubt its suitability for the domain.

### Mitigation

I will prove the concept is sound incrementally by building out an example implementation.

## Git is not an API

Git’s initial deployment and usage model was as a set of command-line binaries, for use by users, and its use in programmatic contexts.

### Mitigation

An example API will be constructed to demonstrate expected usage patterns using [libgit2](https://libgit2.org/). This, with test cases, will form the initial deployment model that can be copied to other domains.

## Increased Storage Costs

Git by default will store files in a repository as both the files laid out (for reading) and in the object store. The retention of history increases the size of the data and makes storing & retrieving subsets less simple.

### Mitigation

Storage is not expensive, and thoughtful repository and file design will minimise the additional costs. The storage of history and the value of being to examine the data’s provenance is worth the expense.

# Future Developments

There likely will be further functionality added to the specification, enabling:

* Directly transferring events into the repository.  
* Sharing subsets of history to a third party.  
* Encrypting subsets of history.

The author’s initial intent is to develop TiG as a library to power a UK-oriented car journaling app, as a proof that the concept is robust and scalable.

The open nature of the data will allow manufacturers to provide their own TiG-powered apps if they wish.

This white paper may later be obsoleted by an RFC or similar (more formal) standard.

The specification should allow specialised LLMs to ingest the data and provide natural language queries.

Ultimately the author hopes that TiG becomes trusted and understood well enough that its usage to store health data becomes a risk-free choice, enabling people to truly take control of their own information.

# Conclusion

TiG is a novel and ambitious project that aims to revolutionise how we store, manage, and transfer personal data. By leveraging the familiar infrastructure of Git and adhering to open standards, TiG offers a robust, secure, and transparent solution for handling sensitive information. 

While there are challenges to overcome, the potential benefits of TiG are significant, particularly in empowering individuals to take control of their data and fostering trust in data exchange processes. With continued development and adoption, TiG could pave the way for a more user-centric and privacy-conscious approach to personal data management.

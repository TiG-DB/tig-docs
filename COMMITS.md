## Commits

Each commit to a TiG database should be covered by a cryptographic signature that describes the source and confirms the date it occurred.
It isn't possible to confirm the absolute "truth" of any event, evidence or telemetry, but we can ensure that timelines haven't
been tampered with.

Signatures use Git's standard [signed tags](https://git-scm.com/docs/git-tag), pointing at the signed commit.

### Signing Convention v1

A signature is a signed annotated tag, created by a signing authority (a *notary*, see below) and stored in the repository
under `refs/tags/tig/sig/`.

#### Tag object

```
object 9fceb02d0ae598e95dc970b74767f19372d61af8
type commit
tag tig/sig/20261020T140000Z
tagger TiG Notary <notary@example.com> 1781298683 +0000

TiG-Signature: 1
Issuer: example.com
Device: SHA256:uNiVztksCsDhcc0u9e8BujQXVUpKZIDTMczCvj3tD2s
-----BEGIN SSH SIGNATURE-----
...
-----END SSH SIGNATURE-----
```

| Field | Description | Format | Example |
| --- | --- | --- | --- |
| `object` | The commit being signed | The full hexadecimal commit hash | 9fceb02d0ae598e95dc970b74767f19372d61af8 |
| `tag` | The tag name | `tig/sig/` followed by the signing time in UTC | tig/sig/20261020T140000Z |
| `tagger` | The notary's name, email and signing time | Standard Git identity, with the time in seconds since the UNIX epoch, always `+0000` | TiG Notary <notary@example.com> 1781298683 +0000 |
| `TiG-Signature` | The signing convention version | Integer | 1 |
| `Issuer` | The entity signing the data | An Internet hostname | example.com |
| `Device` | The device key that requested the signature | The key's SHA-256 fingerprint, as printed by `ssh-keygen -l` | SHA256:uNiVztksCsDhcc0u9e8BujQXVUpKZIDTMczCvj3tD2s |

The signature is an [SSH signature](https://github.com/openssh/openssh-portable/blob/master/PROTOCOL.sshsig) made with an Ed25519
key in the `git` namespace, i.e. what `git tag -s` produces with `gpg.format=ssh`. It covers every line above it, so the
commit, the time and the issuer are all signed. Other lines may be added to the message before the signature; they are signed too,
and must be ignored by readers that don't understand them.

#### Device keys

Each client (e.g. `tig-agent`) generates its own Ed25519 key pair on first run and manages it entirely itself. There are no accounts and nothing is issued by the notary.

* The private key never leaves the device and is never stored in the repository.
* The public key is committed to the repository in `.tig/allowed_signers` (allowed signers format), so that anyone holding the repository can verify its commits.
* Every commit is signed by the device key, using Git's standard SSH commit signing (`git commit -S` with `gpg.format=ssh`).
* A device key is pseudonymous: it identifies a device, not a person or organisation.

#### Signing

1. The client commits locally (signed with its device key), then sends a signing request to the notary. The request contains the full commit hash, the current time and the device's public key, and is itself signed by the device key in the `tig-notary` namespace.
2. The notary checks the request signature and that the request time is within a few minutes of its own clock, so requests can't be replayed.
3. The notary builds the tag object itself, using its own clock for the tag name and `tagger` time, adds the device key's fingerprint as `Device`, signs it and returns the complete tag object.
4. The client verifies the returned tag, then stores it unmodified (e.g. with libgit2's `git_tag_create_from_buffer`).

The notary must never sign a tag object built by a client, otherwise the client could choose the timestamp.
The notary does not need access to the repository: it attests that a commit hash existed at a given time, not what the commit contains.

A commit's hash covers its parent's hash, so a signature on a commit also covers all of its history. Commits need not be signed
individually; clients should sign at least once per sync with a remote, or daily. A signature's time means the commit, and its
history, existed *no later than* that time. Clients may also sign any time before handing a repository to a new owner.

Clients that are offline queue commit hashes and request signatures when they reconnect.

#### Public keys

Any signing relies on a private/public key pair and an authority.

Public keys should be available for confirmation in the [Well Known](https://en.wikipedia.org/wiki/Well-known_URI) folder `tig-public-key`
path of the host used to sign, e.g. `https://example.com/.well-known/tig-public-key`. The file is in Git's
[allowed signers](https://man.openbsd.org/ssh-keygen#ALLOWED_SIGNERS) format, with the issuer's hostname as the principal:

```
example.com namespaces="git",valid-after="20261001" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
```

When a key is rotated, the old key is kept with a `valid-before` date and never removed, so that old signatures can still be checked.

#### Verifying

1. Read the `Issuer` from the tag message and fetch `https://<issuer>/.well-known/tig-public-key`.
2. Run `git -c gpg.ssh.allowedSignersFile=<file> verify-tag <tag>`.
3. Confirm that the signing principal matches the `Issuer`, and that the tag's `object` is the commit being checked or one of its descendants.

This needs nothing beyond standard Git and OpenSSH.

#### What a signature proves

| Claim | Proven by | Not proven |
| --- | --- | --- |
| The commit and its history existed no later than the tag time | The notary's tag signature | That the contents are true |
| The notary is `Issuer` | The key published over HTTPS at `Issuer` | Anything once the domain changes hands: a later owner could publish new keys |
| The commits were made by a particular device | The commit signatures, checked against `.tig/allowed_signers` | Who owns or uses the device |
| That device asked for the notary signature | The `Device` line, which the notary only adds after checking the request signature | |

Because the device's private key is not in the repository, a copy of the repository cannot be continued under the original device's key.
A cloned or stolen history will show new commits that are unsigned, or signed by a key the notary has never seen for that history.
This does not hold if the device's key itself is copied, e.g. from a full disk image.

#### Display

Signature tags appear as labels on their commits in Git clients and hosts. To hide them from `git log`, set:

```sh
git config log.excludeDecoration refs/tags/tig/
```

Git hosts such as GitHub may mark these tags as "Unverified", because the notary's key isn't attached to an account on that host.
That doesn't affect their validity.

### History

An earlier draft put a JWT (`iss`, `iat`, `cha`) in the commit description. It was replaced because a commit cannot contain a
signature of its own hash.

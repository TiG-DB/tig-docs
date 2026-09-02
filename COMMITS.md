## Commits

Each commit to a TiG database should contain a cryptographic signature to describe the source and confirm the date it occurred.
It isn't possible to confirm the absolute "truth" of any event, evidence or telemetry, but we can ensure that timelines haven't
been tampered with.

### Signing Convention v1

The first format for a commit signature is a [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token), with the following fields:

| Key | Name | Description | Format | Example |
| --- | --- | --- | --- | --- |
| `iss` | Issuer | The entity signing the data | An Internet hostname | example.com |
| `iat` | Issued At | The timestamp that the JWT was issued  | Number of seconds since the UNIX epoch (UTC) | 1781298683 |
| `cha` | Commit Hash | The commit hash that is being signed | A hexadecimal SHA hash, with a minium of 16 characters | 00e8bb1093b9f0f42 |

Any signing relies on a private/public key pair and an authority. 

Public keys should be available for confirmation in the [Well Known](https://en.wikipedia.org/wiki/Well-known_URI) folder `tig-public-key` 
path of the host used to sign, e.g. `https://example.com/.well-known/tig-public-key`

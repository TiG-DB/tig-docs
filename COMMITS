## Commits

Each commit to a TiG database should contain a cryptographic signature to describe the source and confirm the date it occurred.
It isn't possible to confirm the absolute "truth" of any event, evidence or telemetry, but we can ensure that timelines haven't
been tampered with.

### Signing Convention v1

The first format for a commit signature is a (JWT)[https://en.wikipedia.org/wiki/JSON_Web_Token], with the following field:
| Key | Name | Description | Format | Example |
| `iss` | `Issuer` | The entity signing the data | An Internet hostname | example.com |
| `iat` | `Issued At` | The timestamp that the JWT was issued  | 1781298683 |
| `cha` | `Commit Hash` | The commit hash that is being signed | 00e8bb1093b9f0f42eacd7ad4ce16122aa9b1d8b |

Any signing relies on a private/public key pair and an authority. 

Public keys should available for confirmation in the `/.well-known/tig-public-key` 
path of the host used to sign, e.g. `https://example.com/.well-known/tig-public-key`

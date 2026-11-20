# Go JOSE

[![godoc](https://pkg.go.dev/badge/github.com/go-jose/go-jose/v5.svg)](https://pkg.go.dev/github.com/go-jose/go-jose/v5)
[![godoc](https://pkg.go.dev/badge/github.com/go-jose/go-jose/v5/jwt.svg)](https://pkg.go.dev/github.com/go-jose/go-jose/v5/jwt)
[![license](https://img.shields.io/badge/license-apache_2.0-blue.svg?style=flat)](https://raw.githubusercontent.com/go-jose/go-jose/main/LICENSE)

Package jose aims to provide an implementation of the Javascript Object Signing
and Encryption set of standards. This includes support for JSON Web Encryption,
JSON Web Signature, and JSON Web Token standards.

## Overview

The implementation follows the
[JSON Web Encryption](https://dx.doi.org/10.17487/RFC7516) (RFC 7516),
[JSON Web Signature](https://dx.doi.org/10.17487/RFC7515) (RFC 7515), and
[JSON Web Token](https://dx.doi.org/10.17487/RFC7519) (RFC 7519) specifications.
Tables of supported algorithms are shown below. The library supports both
the compact and JWS/JWE JSON Serialization formats, and has optional support for
multiple recipients. It also comes with a small command-line utility
([`jose-util`](https://pkg.go.dev/github.com/go-jose/go-jose/v5/jose-util))
for dealing with JOSE messages in a shell.

Older versions of this package used a forked version of `encoding/json`, but v5
uses `encoding/json/v2`. It continues to have case-sensitive matching and
rejection of duplicate field names.

### Multi-recipient JWE structure

JWE JSON Serialization has two syntaxes, and this package keeps their structural
boundary strict in both directions:

* A JWE with exactly one recipient is serialized with the **flattened** syntax
  (RFC 7516 Section 7.2.2). Its top-level members include `header` and
  `encrypted_key`.
* A JWE with more than one recipient is serialized with the **general** syntax
  (RFC 7516 Section 7.2.1). Its top-level members are `protected`,
  `unprotected`, `recipients`, `aad`, `iv`, `ciphertext`, and `tag`. `header`
  and `encrypted_key` are deliberately **not** emitted at the top level: every
  recipient's header and encrypted key appear only in that recipient's own entry
  of the `recipients` array, so the owner of each encrypted key is explicit.
* On input, a message that combines a `recipients` array with a top-level
  `header` or `encrypted_key` member is rejected with
  `ErrConflictingRecipientMembers`. RFC 7516 Section 7.2.2 makes those members
  mutually exclusive; silently discarding the flattened member would leave the
  ownership of an encrypted key unverifiable. This error is distinct from the
  errors reported for genuinely missing or invalid headers.

Compact serialization is unaffected: it still supports only single-recipient
objects and returns `ErrNotSupported` for multi-recipient ones.

#### Migration note for older issuers

Messages produced by older versions of this package (and by other
implementations) may carry a top-level `encrypted_key` or `header` alongside a
`recipients` array. The key at the top level is a copy of the first recipient's
encrypted key; the raw per-recipient entries in the `recipients` array remain
authoritative. Because such messages put the audit-relevant encrypted key in two
places at once, they are now rejected at parse time rather than being quietly
reinterpreted.

To interoperate during the rollout, re-issue or rewrite affected messages so each
encrypted key appears only inside its recipient entry: parse with an older
release, drop the top-level `header`/`encrypted_key` members, and re-serialize
(the general syntax produced by this release never reintroduces them). The
alternative — silently preferring the `recipients` entries and dropping the
top-level duplicate — was rejected because it makes the two readings
indistinguishable to auditors and hides real disagreements between the duplicated
fields. Accepting the mixed form only for messages whose top-level copy is
byte-identical to the first recipient's key was also rejected: it adds a
heuristic ownership rule and keeps two sources of truth in circulation
indefinitely.

### Versions

Version 5 is in development on the main branch of this repository.  It will be
released with several breaking API changes, and will require Go's
`encoding/json/v2` and Go 1.27.  In the future it will support at least the
current and previous Go release.

Version 4 is now feature-frozen, receiving only bug fixes.

    import "github.com/go-jose/go-jose/v4"

It supports at least the current and previous Go release.
Version 4 will be supported until Go 1.29 is released in August 2027.

Version 3 is only receiving critical security updates. Migration to Version 4 is recommended.
Version 3 will no longer be supported once Go 1.28 is released in February 2027.

Versions 1 and 2 are obsolete, but can be found in the old repository, [square/go-jose](https://github.com/square/go-jose).

### Supported algorithms

See below for a table of supported algorithms. Algorithm identifiers match
the names in the [JSON Web Algorithms](https://dx.doi.org/10.17487/RFC7518)
standard where possible. The Godoc reference has a list of constants.

| Key encryption         | Algorithm identifier(s)                        |
|:-----------------------|:-----------------------------------------------|
| RSA-PKCS#1v1.5         | RSA1_5                                         |
| RSA-OAEP               | RSA-OAEP, RSA-OAEP-256                         |
| AES key wrap           | A128KW, A192KW, A256KW                         |
| AES-GCM key wrap       | A128GCMKW, A192GCMKW, A256GCMKW                |
| ECDH-ES + AES key wrap | ECDH-ES+A128KW, ECDH-ES+A192KW, ECDH-ES+A256KW |
| ECDH-ES (direct)       | ECDH-ES[^multirecipient]                       |
| Direct encryption      | dir[^multirecipient]                           |

[^multirecipient]: Not supported in multi-recipient mode

| Signing / MAC     | Algorithm identifier(s)                     |
|:------------------|:--------------------------------------------|
| RSASSA-PKCS#1v1.5 | RS256, RS384, RS512                         |
| RSASSA-PSS        | PS256, PS384, PS512                         |
| HMAC              | HS256, HS384, HS512                         |
| ECDSA             | ES256, ES384, ES512                         |
| Ed25519           | EdDSA                                       |
| ML-DSA            | ML-DSA-44, ML-DSA-65, ML-DSA-87             |

| Content encryption | Algorithm identifier(s)                     |
|:-------------------|:--------------------------------------------|
| AES-CBC+HMAC       | A128CBC-HS256, A192CBC-HS384, A256CBC-HS512 |
| AES-GCM            | A128GCM, A192GCM, A256GCM                   |

| Compression        | Algorithm identifiers(s) |
|:-------------------|--------------------------|
| DEFLATE (RFC 1951) | DEF                      |

### Supported key types

See below for a table of supported key types. These are understood by the
library, and can be passed to corresponding functions such as `NewEncrypter` or
`NewSigner`. Each of these keys can also be wrapped in a JWK if desired, which
allows attaching a key id.

| Algorithm(s)       | Corresponding types                                                                                                                  |
|:-------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| RSA                | *[rsa.PublicKey](https://pkg.go.dev/crypto/rsa/#PublicKey), *[rsa.PrivateKey](https://pkg.go.dev/crypto/rsa/#PrivateKey)             |
| ECDH, ECDSA        | *[ecdsa.PublicKey](https://pkg.go.dev/crypto/ecdsa/#PublicKey), *[ecdsa.PrivateKey](https://pkg.go.dev/crypto/ecdsa/#PrivateKey)     |
| EdDSA<sup>1</sup>  | [ed25519.PublicKey](https://pkg.go.dev/crypto/ed25519#PublicKey), [ed25519.PrivateKey](https://pkg.go.dev/crypto/ed25519#PrivateKey) |
| ML-DSA<sup>2</sup> | *[mldsa.PublicKey](https://pkg.go.dev/crypto/mldsa#PublicKey), *[mldsa.PrivateKey](https://pkg.go.dev/crypto/mldsa#PrivateKey)       |
| AES, HMAC          | []byte                                                                                                                               |

## Examples

[![godoc](https://pkg.go.dev/badge/github.com/go-jose/go-jose/v5.svg)](https://pkg.go.dev/github.com/go-jose/go-jose/v5)
[![godoc](https://pkg.go.dev/badge/github.com/go-jose/go-jose/v5/jwt.svg)](https://pkg.go.dev/github.com/go-jose/go-jose/v5/jwt)

Examples can be found in the Godoc
reference for this package. The
[`jose-util`](https://github.com/go-jose/go-jose/tree/main/jose-util)
subdirectory also contains a small command-line utility which might be useful
as an example as well.

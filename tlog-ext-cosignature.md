# Transparency Log Extended Cosignatures (DRAFT)

An extended cosignature is a statement by some party in a transparency log
ecosystem that it verified the consistency of a [checkpoint][], along with some
other properties specified by that cosigner. Log clients can verify a quorum of
extended cosignatures to provide split-view attacks and also obtain assurance of
other properties of some log entry.

```
example.com/behind-the-sofa
20852163
CsUYapGGPo4dkMgIAUqom/Xajj7h2fB2MPA3j2jxq2I=

— example.com/behind-the-sofa Az3grlgtzPICa5OS8npVmf1Myq/5IZniMp+ZJurmRDeOoRDe4URYN7u5/Zhcyv2q1gGzGku9nTo+zyWE+xeMcTOAYQ8=
— witness.example.com/w1 jWbPPwAAAABkGFDLEZMHwSRaJNiIDoe9DYn/zXcrtPHeolMI5OWXEhZCB9dlrDJsX3b2oyin1nPZ\nqhf5nNo0xUe+mbIUBkBIfZ+qnA==
— mirror.example.com/m1 jWbPPwAAAABkGFDLEZMHwSRaJNiIDoe9DYn/zXcrtPHeolMI5OWXEhZCB9dlrDJsX3b2oyin1nPZ\nqhf5nNo0xUe+mbIUBkBIfZ+qnA==
```

## Conventions used in this document

Data structures are defined according to the conventions laid out in Section 3
of [RFC 8446][].

`U+` followed by four hexadecimal characters denotes a Unicode codepoint, to be
encoded in UTF-8. `0x` followed by two hexadecimal characters denotes a byte
value in the 0-255 range. `||` denotes concatenation.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [BCP 14][] [RFC 2119][] [RFC
8174][] when, and only when, they appear in all capitals, as shown here.

[RFC 8446]: https://www.rfc-editor.org/rfc/rfc8446.html
[BCP 14]: https://www.rfc-editor.org/info/bcp14
[RFC 2119]: https://www.rfc-editor.org/rfc/rfc2119.html
[RFC 8174]: https://www.rfc-editor.org/rfc/rfc8174.html
[RFC 8032]: https://www.rfc-editor.org/rfc/rfc8032.html

## Introduction

An extended cosignature is a generalization of a [witness][] [cosignature][].
Entities that create extended cosignatures are known as **extended cosigners**.
They may also be referred to as **cosigners** when the formulation is clear
from context.

Extended cosigners have an **name** and a public key. The name is a unique
identifier for the cosigner. The name MUST be non-empty, and it SHOULD be
a schema-less URL containing neither Unicode spaces nor plus (U+002B), such
as `example.com/mirror42`. This is only a recommendation to avoid collisions,
and clients MUST NOT assume that the name is following this format or that
the URL corresponds to a reachable endpoint.

For each supported log, an extended cosigner follows some append-only branch of
the log, either by being the log operator or by checking consistency proofs, as
in a witness. If it makes an extended cosignature for some checkpoint, it
asserts that this checkpoint is part of its append-only branch. The signature
also provides additional cosigner-specific assertions about the checkpoint.

For example, a [mirror][] cosignature asserts that the checkpoint's contents are
available from its monitoring interface. A [witness][] MAY also be operated as
an extended cosigner, whose signatures do not provide any assertions beyond the
baseline append-only property. Other documents MAY define extended cosigner
roles that provide other assertions, e.g. checking some [checkpoint][]
extension, or some property of the entries.

When an extended cosigner signs checkpoints, it is held responsible *both*
for upholding the append-only property *and* for meeting its defined guarantees
for all entries in any checkpoints that it signed.

A single extended cosigner, with a single cosigner name and public key, MAY
generate extended cosignatures for checkpoints from multiple logs. The signed
message, defined below, includes both the cosigner name and log origin.

An extended cosigner's name identifies the cosigner and thus the assertions
provided. If a single operator performs multiple extended cosigner roles in an
ecosystem, each role MUST use a distinct cosigner name and SHOULD use a
distinct key.

## Format

Concretely, an extended cosignature is a [note signature][] applied to a
[checkpoint][]. The note signature's key name MUST be the extended cosigner's
name.

The key ID MUST be

    SHA-256(<name> || "\n" || 0x06 || 32-byte Ed25519 cosigner public key)[:4]

Clients are configured with tuples of (cosigner name, public key, supported
extended cosignature version). Based on that, they can compute the expected
name and key ID, and ignore any signature lines that don't match.

Future extended cosignature formats MAY reuse the same cosigner public key with
a different key ID algorithm byte (and a different signed message header line).

The signature MUST be a 72-byte `timestamped_signature` structure.

    struct timestamped_signature {
        u64 timestamp;
        u8 signature[64];
    }

"timestamp" is the time at which the cosignature was generated, as seconds since
the UNIX epoch (January 1, 1970 00:00 UTC).

"signature" is an Ed25519 ([RFC 8032][]) signature from the cosigner public key
over the message defined in the next section.

## Signed message

The signed message MUST be three newline (U+000A) terminated header lines
followed by the whole note body of the cosigned checkpoint (including the final
newline, but not including any signature lines).

The header lines are:

* The fixed string `ext-cosignature/v1` for domain separation
* The extended cosigner name
* The fixed string `time`, a single space (0x20), and the number of seconds since the UNIX epoch encoded as an ASCII decimal with no leading zeroes

The timestamp value in the third header line MUST match
`timestamped_signature.timestamp` in the signature.

Semantically, a v1 extended cosignature is a statement that, as of the specified
time, the specified checkpoint is one of the largest size which:

* has a tree hash which is consistent with all other checkpoints signed by the named extended cosigner
* satisfies all other properties asserted by the named extended cosigner

Extension lines MAY be included in the checkpoint by the log, and if present
MUST be included in the cosigned message. However, no semantic statement is made
about any extension line, unless the extended cosigner is defined to make them.

[checkpoint]: https://c2sp.org/tlog-checkpoint
[cosignature]: https://c2sp.org/tlog-cosignature
<!-- TODO: Replace this link with https://c2sp.org/tlog-mirror -->
[mirror]: ./tlog-mirror.md
[note signature]: https://c2sp.org/signed-note
[witness]: https://c2sp.org/tlog-witness

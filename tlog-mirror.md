# Transparency Log Mirrors (DRAFT)

This document describes how to mirror a transparency log, and how to obtain
signatures asserting that a mirror has done so.

<!-- TODO: Replace these links with https://c2sp.org/tlog-ext-cosignature -->
[extended cosigner]: ./tlog-ext-cosignature.md
[extended cosignature]: ./tlog-ext-cosignature.md
[checkpoint]: https://c2sp.org/tlog-checkpoint
[tiled transparency log]: https://c2sp.org/tlog-tiles
[witness]: https://c2sp.org/tlog-witness
[percent-encoded]: https://www.rfc-editor.org/rfc/rfc3986.html#section-2.1

## Conventions used in this document

`U+` followed by four hexadecimal characters denotes a Unicode codepoint, to be
encoded in UTF-8. `0x` followed by two hexadecimal characters denotes a byte
value in the 0-255 range.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [BCP 14][] [RFC 2119][] [RFC
8174][] when, and only when, they appear in all capitals, as shown here.

[RFC 4648]: https://www.rfc-editor.org/rfc/rfc4648.html
[BCP 14]: https://www.rfc-editor.org/info/bcp14
[RFC 2119]: https://www.rfc-editor.org/rfc/rfc2119.html
[RFC 8174]: https://www.rfc-editor.org/rfc/rfc8174.html
[RFC 6962]: https://www.rfc-editor.org/rfc/rfc6962.html

## Introduction

A mirror is an [extended cosigner][] that stores a copy of a log and, when
signing a [checkpoint][], provides the additional guarantee that the mirror has
durably logged the contents of the checkpoint and has made it accessible.

A mirror is defined by a cosigner origin, a pubic key, and by two URL prefixes:
the *submission prefix* for write APIs and the *monitoring prefix* for read
APIs. A mirror MAY use the same value for both the *submission prefix* and the
*monitoring prefix*.

For each supported origin log, the mirror is configured with:

* The log's public key
* The log's URL prefix
* An optional list of monitoring prefixes for other mirrors for the log

The mirror maintains a copy of each origin log and serves it publicly via the
[tiled transparency log][] interface. It uses a URL prefix of
`<monitoring prefix>/<encoded origin>`, where `encoded origin` is the log's
origin, [percent-encoded][]. The checkpoint served from this prefix MUST include
an [extended cosignature][] from the mirror.

## Updating a Mirror

To facilitate updating, each mirror maintains a witness checkpoint for each
origin log. The witness checkpoint is at or ahead of the mirrored checkpoint and
tracks data that has yet to be mirrored.

Mirrors update in two stages:

1. Update the witness checkpoint by verifying a signed checkpoint and
   consistency proof.
2. Update the mirror checkpoint to the witness checkpoint by downloading tiles.

This two-stage design minimizes atomic operations when operating a mirror. The
mirror checkpoint updates asynchronously from the witness checkpoint, so the
witness checkpoint can continue to be updated while the tiles are downloaded for
the mirror checkpoint.

### Witness Checkpoint

The mirror implements a [witness][]'s `add-checkpoint` endpoint to update its
witness checkpoint and schedule an update:

    POST <submission prefix>/add-checkpoint

The request is handled identically to that of a witness, updating the witness
checkpoint (but not the mirror checkpoint), with the exception that it does not
need to generate and respond with any cosignatures. The mirror MAY handle the
request by internally updating the witness checkpoint and responding with an
empty response body.

The mirror MAY maintain respond with witness cosignatures if it wishes to
additionally provide a public witness service using its witness state. If so,
this witnessing service MUST have a name that is different from the mirror's
cosigner origin.

In addition to updating based on external requests to the `add-checkpoint`
endpoint, a mirror SHOULD also periodically poll the origin log, and other
mirrors, for updates. It does so by downloading the log's latest checkpoint and,
if it is ahead of the mirror's witness position, downloading a consistency proof
and then updating the witness position as in `add-checkpoint`. This process MAY
be implemented externally to the mirror by posting the result to
`add-checkpoint`.

### Mirror Checkpoint

Whenever the witness checkpoint is ahead of the mirror checkpoint for some
supported log, the mirror downloads tiles to update the mirror checkpoint. The
mirror MUST NOT update the mirror checkpoint and generate an
[extended cosignature][] until all tiles contained in the new checkpoint value
are downloaded and available.

Not all witness checkpoints will necessarily have corresponding to mirror
checkpoints. The mirror's update process MAY skip witness checkpoints, e.g. if
the witness checkpoints update more frequently. For example, suppose the witness
checkpoint is first updated to tree size 100, then 200, then 300. If the updates
to 200 and 300 occur while the mirror is downloading tiles for tree size 100,
the mirror checkpoint may skip tree size 200 and next update to 300. Although a
tree size of 300 contains all entries from a tree size of 200, some
[partial tiles][tiled transparency log][] may not overlap.

#### Recommended Update Procedure

This section gives a RECOMMENDED procedure for updating the mirror checkpoint.
It saves tiles in an order such that each tile is authenticated to the witness
position before being saved to the mirror's tile store. This ensures:

* There is no need to run the procedure atomically with other log operations.
  Only storing individual resources must be atomic.

* The procedure can be safely interrupted and restarted at any time, or even run
  concurrently with other instances of the procedure.

When a mirror's witness position is updated, the mirror schedules a job to
run the following procedure. If this job is already scheduled, the mirror SHOULD
NOT schedule a redundant job, though doing so will not impact correctness of
mirror.

1. Let `mirror_size` be the tree size of the mirror checkpoint, or zero if
   the mirror has not copied the log yet.

2. Let `witness_checkpoint` be the witness checkpoint. Let `witness_size` be its
   tree size. Note the witness checkpoint may change while this procedure is
   running.

3. If `mirror_size` is equal to `witness_size`, exit this procedure.

4. Determine the partial tiles contained in a tree of size `witness_size`. There
   is at most one per level, at the right-most position.

5. Check if each such tile is already in the tile store. If any are missing:

   1. Download each of those tiles to memory.
   2. Reconstruct the right edge of the tree from these tiles and compute the
      root hash. Note that every entry of these files will be incorporated into
      the root hash.
   3. Compare the root hash to `witness_checkpoint`'s hash. If it matches, save
      each downloaded tile to the tile store. If it does not match, terminate
      this procedure with an error.

6. Determine the full tiles contained in a tree of size `witness_size` that are
   not contained in a tree of size `mirror_size`.

7. For each tile that is not already in the tile store, run the following to
   download the tile. Tiles MAY be downloaded in parallel, or in any order,
   provided that parent tiles, full or partial, are checked and stored before
   checking a child tile. Downloading tile in order of decreasing level achieves
   this.

   1. Download the tile to memory.
   2. Compute the hash of a Merkle Tree built over the 256 entries in the tile.
   3. Compare the hash to the corresponding entry in the parent tile.
   4. If it matches, save the downloaded tile to the tile store. If it does not,
      terminate this procedure with an error.

8. Determine the entry bundles contained in a tree of size `witness_size` that
   are not contained in a tree of size `mirror_size`.

9. For each entry bundle that is not already in the tile store, run the
   following to download the bundle. Bundles MAY be downloaded in parallel, or
   in any order.

   1. Download the entry bundle to memory.
   2. Compute the hashes of each entry in the bundle.
   3. Check that each hash matches the corresponding tile at level 0.
   4. If all hashes match, save the downloaded bundle to the tile source. If any
      hash does not, terminate this procedure with an error.

10. After all new tiles and bundles have been verified and persisted to storage,
    run the following step atomically. This step MUST be synchronized with other
    changes to the mirror state:

     1. Check if the mirror checkpoint's tree size is still less than
        `witness_size`.
     2. If so, add an extended cosignature to `witness_checkpoint` and save the
        result as the new mirror checkpoint.

A mirror following this procedure MAY eagerly download resources to memory, or
some local cache, in a different order than recommended. However, those
resources will not have been validated. This procedure is designed to allow a
mirror to update while bounding the number of unvalidated resources.

A mirror MAY opt to skip downloading a parent tile in favor of computing it
based on its children tiles from a lower level. However, those children tiles
cannot be validated until the parent tile is validated. Such a strategy thus
requires the update process to mantain a larger number of unvalidated resources.

#### Downloading Resources

Mirrors MAY obtain tiles and bundles from any source, including:

* the origin log's HTTP interface
* another mirror's HTTP interface
* a local cache of not-yet-validated resources
* uploaded through some other HTTP endpoint
* computed from child tiles

However, the mirror MUST validate all received resources are consistent with the
new mirror checkpoint before signing the new checkpoint. It is RECOMMENDED to
validate new sources before persisting them, as in the
[recommended update procedure](#recommended-update-procedure).

As in the origin log, when the mirror validates and stores a full tile, it MAY
delete the corresponding partial tiles. However, it MUST NOT do this until the
full tile has been validated.

When downloading partial tiles, the mirror MAY fetch the corresponding full tile
as a fallback. However, if it does so, it MUST truncate the result to the size
of the requested partial tile. Extra entries will not be validated by the
checkpoint.

### TBD Push-based Updates

TODO: It may be useful to have a push-based update endpoint, so that a log
can upload data to a mirror, and get a response once the mirror has updated far
enough. In the current design, the log needs to poll the checkpoint endpoint to
find out when the mirror has caught up.

It would be a matter of posting a stream of binary data containing
the necessary tiles in order of the
[recommended update procedure](#recommended-update-procedure). The mirror can
then run that procedure but, instead of downloading tiles, pull data off the
stream. (It is possible the stream contains data you already have. That's fine,
just skip it.) This is where being able to run the update process concurrently
is a real win.

The main challenge is synchronizing with *some* known witness position. The
client would POST something like `<old_size> <new_size>` followed by the data
for such an update. `old_size` must be less than or equal to the mirror
position. (If too far behind, the mirror might send a 409 Conflict and tell you
to be less wasteful.) `new_size` obviously cannot be greater than the witness
position. If it's equal, we're golden. If less than, we need to be told the hash
and a consistency proof to the actual witness position... more likely we'd ask
for a 409 Conflict, but then you might never be able to get in sync if the
witness position updates too fast.

A push-based endpoint design could either subsume the `add-checkpoint` endpoint,
or be done in two separate HTTP requests, one to update the witness position and
another to stream data.

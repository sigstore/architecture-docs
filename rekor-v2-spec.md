# Rekor v2, a Tiles-based Transparency Service

## 1. Introduction

Rekor is a cryptographically verifiable transparency log for signing events.
Rekor enables software maintainers to record the metadata of a signing event to
a tamper-evident, verifiable, append-only public record, allowing other parties
to verify the integrity and provenance of software artifacts.

Rekor v2 represents a major architectural shift from [v1](./rekor-spec.md). It
introduces a tiles-based log structure and API for improved performance,
scalability, and operational simplicity.

### 1.1 Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14
\[[RFC2119](https://www.rfc-editor.org/info/rfc2119)\]
\[[RFC8174](https://www.rfc-editor.org/info/rfc8174)\] when, and only when, they
appear in all capitals, as shown here.

## 2. Overview

Sigstore provides a transparency service to serve as an auditable public ledger
of software signing events. The aim of a signature transparency log is to allow
interested parties to audit signature issuance and force key compromises into
the open. Rekor v2 performs most of the same role in the system as [Rekor
v1](./rekor-spec.md) and functions in a similar way, i.e., records contain the
same metadata and the log is based on a Merkle tree structure.

Rekor v2 provides a protocol for signing clients to submit entries to the log
and receive inclusion proofs as a response that can be verified offline. It also
exposes the underlying tile structure as a cacheable read API that can be used for
verifying the consistency of the Merkle tree, computing inclusion proofs, and
fetching raw entries.

## 3. Terminology

| Term | Definition |
| :---- | :---- |
| Merkle Tree | Cryptographic data structure in which every leaf node is represented by a cryptographic hash, and every parent node is the cryptographic hash of the concatenation of the children nodes. Allows for efficient proofs of inclusion and consistency. |
| Log ID | The checkpoint key ID as defined in the [Signed Note](https://github.com/C2SP/C2SP/blob/main/signed-note.md#signatures) C2SP spec. |
| Log Index | Integer index assigned to each entry in a log shard. Monotonically increasing for a given shard, but restarts from zero for new shards.  |
| Inclusion Proof | Proof that a given leaf value is committed to by a given Checkpoint. Consists of a list of ordered hashes of size log(N) where N is the size of the shard. |
| Consistency Proof | Proof that a given log is a prefix of another log, i.e. append-only. Consists of a list of ordered hashes. |
| Checkpoint | Commitment by the log to its current state, containing the Merkle tree root hash and signed by Rekor's private key. |
| Shard | An instance of the log. A new instance of the log is created on a regular interval to prevent the log from indefinitely growing, for scalability, as well as for the opportunity to rotate the checkpoint signing key. Shards are fully independent from one another, and clients use metadata distributed by TUF to discover new log shards. |
| Witness | An entity which maintains its view of a "latest consistent checkpoint" from a log. May sign and expose the checkpoints it witnesses. |
| Synchronous Witnessing | A [protocol](https://github.com/C2SP/C2SP/blob/main/tlog-witness.md) by which a transparency log requests checkpoint cosignatures from a witness network in order to produce self-contained inclusion proofs that can be returned to clients and verified offline. |
| Monitor | An entity which continually observes the contents of a log to look for use of a user's key or identity and alerts the user to its potential compromise. Referred to as a Verifier in the [claimant model](https://github.com/sigstore/community/tree/main/docs/claimantmodel#rekor-identity-based-signature).
| Tile | Concatenated sequences of consecutive Merkle Tree Hashes at a certain tree height. Stored and served as highly cacheable objects, so clients wishing to fetch hashes for a consistency proof or inclusion proof can fetch whole tiles rather than request individual hashes. |

## 4. Changes from V1

### 4.1 Offline Verification

Clients that need to verify the inclusion of a given signing event SHOULD do so
using using offline verification material provided by the signing client and
without performing an online query of the transparency log. It is possible, by
fetching tiles, to compute an inclusion proof given only a log index, but this
is discouraged and unnecessary.

### 4.2 Timestamps

Rekor v2 does not act as a timestamping authority and Signed Entry Timestamps
will not be returned as a response to an uploaded entry. Signing clients SHOULD
fetch a timestamp from an [RFC
3161](https://www.ietf.org/rfc/rfc3161.txt)-conformant Timestamp Authority and
include it as part of their verification material in order for verifying clients
to verify Fulcio-issued ephemeral signing certificates.

### 4.3 Types

Rekor v1 supported a wide variety of "pluggable" entry types to specify how the
transparency service should parse a request to get an entry or its digest, an
entry's signature, and the signature verifier. Rekor v2 has reduced the number
of supported entry types to the
[HashedRekord v0.0.2](https://github.com/sigstore/rekor-tiles/blob/main/api/proto/rekor/v2/hashedrekord.proto)
type, for basic artifact signing, and the
[DSSE v0.0.2](https://github.com/sigstore/rekor-tiles/blob/main/api/proto/rekor/v2/dsse.proto)
type, for attestations. Clients are responsible for formatting their entry
metadata into one of these two types.

### 4.4 Batching and Checkpoint Intervals

Rekor v2 internally batches entry submissions and integrates them as a group,
and updates the checkpoint periodically. This means that an individual
submission may have a delay of up to 10 seconds in receiving a response, but on
average a batch of submissions should have a faster integration period. Stable
checkpoint publishing is no longer needed because the infrequency with which the
checkpoint is updated solves the instability issue for potential witnesses
needing to reach a consensus on the consistency of a log.

### 4.5 Search Index

Rekor v2 does not support the search API that Rekor v1 provided. The results or
absense of results for a search query were not verifiable, meaning mainly that
it was possible for a log to lie by pretending that no entry existed for a given
query. A separate, independent service will be implemented in the future to fill
the void in this functionality.

## 3. Architecture

### 3.1 Tiles

Storage of the Merkle tree is optimized by organizing it into
[tiles](https://research.swtch.com/tlog#tiling_a_log). The internal tile layout
is exposed as a readable API in order for clients to query for the hashes needed
to compute a consistency proof. The static nature of tiles means that it is
possible to serve the read API directly from a file or object service rather
than as a routable path through the Rekor service, and that it can easily be
cached in a CDN or in clients' local caches.

### 3.2 Sharding

In order to rotate the transparency log's checkpoint signing key and to keep the
size of the tree manageable, a new tree is periodically created and clients
transition to uploading entries to the new tree. This is called "sharding" the
log.

In Rekor v1, the shards were abstracted behind a common URL and a global index
counter. In Rekor v2, shards are fully independent and clients must discover,
via TUF, the correct shard to upload to and the correct key to verify an
inclusion proof with.

### 3.3 Witnessing

Rekor v2 can be configured to subscribe to a network of log witnesses. Upon
creating a new checkpoint, it sends the checkpoint to witnesses for
counter-signatures, and blocks until it receives the co-signed checkpoint which
it then returns to signing clients. Verifying clients SHOULD verify the
co-signatures against the keys in their trust root for the witnesses.

Synchronous witnessing like this ensures that consistency verification becomes
an automatic client concern. The converse, asynchronous witnessing, in which it
is the client's responsibility to gather witness cosignatures on a checkpoint,
meant that users could treat it as an afterthought.

## 4. API

### 4.1 Records

#### 4.1.1 Requests

An entry is submitted to the log as either a
[HashedRekordRequestV002](https://github.com/sigstore/rekor-tiles/blob/f3cba09c2f92f1d2d5a7ca2b5694b66941a82d41/api/proto/rekor/v2/hashedrekord.proto#L31)
request or a
[DSSERequestV002](https://github.com/sigstore/rekor-tiles/blob/f3cba09c2f92f1d2d5a7ca2b5694b66941a82d41/api/proto/rekor/v2/dsse.proto#L32C9-L32C24)
request. These are canonicalized by the log service in order to deduplicate
equivalent entries. This is done by converting the requests to their
corresponding entry types
([HashedRekordLogEntryV002](https://github.com/sigstore/rekor-tiles/blob/f3cba09c2f92f1d2d5a7ca2b5694b66941a82d41/api/proto/rekor/v2/hashedrekord.proto#L38)
and
[DSSELogEntryV002](https://github.com/sigstore/rekor-tiles/blob/f3cba09c2f92f1d2d5a7ca2b5694b66941a82d41/api/proto/rekor/v2/dsse.proto#L40C9-L40C25))
and marshalling the entry protobuf message as JSON and hashing the result.

#### 4.1.2 Responses

The response to submitting an entry is a
[TransparencyLogEntry](https://github.com/sigstore/protobuf-specs/blob/4df5baadcdb582a70c2bc032e042c0a218eb3841/protos/sigstore_rekor.proto#L94),
which contains a verifiable record of a signing event and includes the necessary
information to validate the signature and the entry's inclusion in the log.

#### 4.1.3 Signature Verification

When the service receives a proposed entry for submission, it must validate the
signing event using the data contained in the record. In general, it will
contain some representation of the data signed, the signature(s), and the key
material used to verify the signature(s).

Validation metadata should include verification material which can verify the
signature(s). Validation metadata will commonly be a code-signing certificate or
a public key.

### 4.2 HTTP and gRPC API

The Rekor v2 write API is accessible over HTTP or gRPC. Refer to the [protobuf
specification](https://github.com/sigstore/rekor-tiles/tree/main/api/proto) for
the exact schema, and the [clients
guide](https://github.com/sigstore/rekor-tiles/blob/main/CLIENTS.md) for
examples. Tiles, entry leaves, and the current checkpoint can be fetched via an
HTTP API, specified in the C2SP spec for
[tiles](https://github.com/C2SP/C2SP/blob/main/tlog-tiles.md#merkle-tree),
[entries](https://github.com/C2SP/C2SP/blob/main/tlog-tiles.md#log-entries), and
[checkpoints](https://github.com/C2SP/C2SP/blob/main/tlog-tiles.md#checkpoints).

### 4.3 Checkpoint Format

The Merkle tree checkpoint, accessible via a GET request to the
`/api/v2/checkpoint` endpoint, follows the [C2SP Transparency Log
Checkpoints](https://github.com/C2SP/C2SP/blob/main/tlog-checkpoint.md)
specification. The checkpoint MAY additionally include witness signatures as
specified in the [Transparency Log Witness
Protocol](https://github.com/C2SP/C2SP/blob/main/tlog-witness.md).

## 5. Client Workflows

### 5.1 Signers

A client signing an artifact uploads the entry record, formatted as either a
[HashedRekordRequestV002](https://github.com/sigstore/rekor-tiles/blob/f3cba09c2f92f1d2d5a7ca2b5694b66941a82d41/api/proto/rekor/v2/hashedrekord.proto#L31C9-L31C32)
or
[DSSERequestV002](https://github.com/sigstore/rekor-tiles/blob/f3cba09c2f92f1d2d5a7ca2b5694b66941a82d41/api/proto/rekor/v2/dsse.proto#L32C9-L32C24) to the
`/api/v2/log/entries` (HTTP) or `dev.sigstore.rekor.v2.Rekor.CreateEntry` (gRPC)
service. The client receives, as a response, a
[TransparencyLogEntry](https://github.com/sigstore/protobuf-specs/blob/4df5baadcdb582a70c2bc032e042c0a218eb3841/protos/sigstore_rekor.proto#L94),
which contains the latest checkpoint and an inclusion proof for the entry.
Because entries are batched before being integrated, and because checkpoints are
published only at regular intervals, the client may have to wait for a few
seconds (on the order of 2-10 seconds) before receiving a response, depending on
the configuration of the service.

### 5.2 Believers

A believer is software installer that has an interest in verifying claims on the
software package. Such a client verifying an entry's inclusion in the log
SHOULD NOT perform an online lookup of the entry. The signing client SHOULD
provide the entry's checkpoint, inclusion proof, verification key and digest in
the verification material it provides for the verifying client. The verifying
client uses the material it is provided with and the public key of the log to
verify the entry's inclusion in the log.

To verify an entry, a client:

1. verifies the signature of the checkpoint against the server's public key and
   the expected server name, using the [checkpoint
   format](https://github.com/C2SP/C2SP/blob/main/tlog-checkpoint.md#note-text)
2. verifies the inclusion proof using the algorithm specified in [RFC 6962
   2.1.1](https://datatracker.ietf.org/doc/html/rfc6962#section-2.1.1)
3. compares the leaf hash with the provided artifact
4. optionally, verifies witness signatures if available

### 5.3 Monitors

A monitor (also called simply a Verifier in the [claimant
model](https://github.com/sigstore/community/tree/main/docs/claimantmodel#rekor-identity-based-signature)
is a client that monitors the contents of a transparency log to protect a given
user against entry key compromise. If a key or identity is found to have been
used in a signing event, it is done so in the open, and a monitor detects it and
reports it to the user to whom the key or identity belongs. The user is
responsible for determining how to use that information, for example, to notify
a software ecosystem's users of the compromise. The monitor uses the tile
entries API at `/api/v2/tile/entries` to tail the log for its contents.

### 5.4 Witnesses

Synchronous witnesses are responsible for verifying the consistency of a log.
Their co-signatures of a checkpoint are returned to the log for distribution to
signing clients to use in their verification material bundle. Witnessing is
described in the [Transparency Log Witness
Protocol](https://github.com/C2SP/C2SP/blob/main/tlog-witness.md).

Verifying the log's consistency is essential for proving that a log remains
append-only and has not been tampered with. Using a diverse network of
corroborating witnesses helps to mitigate split-view attacks, where a malicious
log could present alternate views of a tree to different parties.

## Privacy

Rekor v2 has most of the same privacy considerations as [Rekor
v1](./rekor-spec.md#6-privacy), such as that sensitive content may be recorded
and publicly viewable in the log and parties uploading entries should be aware
of this potential exposure. Rekor v2 has a privacy advantage over Rekor v1 in
that offline verification is the gold standard and has a better well-lit path than
online verification, so verifying clients who follow the recommendations have no
opportunity to expose the entry they are interested in verifying to the log
service.

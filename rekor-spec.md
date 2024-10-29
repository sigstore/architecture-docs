# Rekor, A Transparency Service

This document describes Rekor, a signature transparency service that securely records and makes verifiable the metadata of signed software artifacts, ensuring trust and integrity in the software supply chain. 

## 1. Introduction

Sigstore is an open source project that provides a secure framework for signing, verifying, and protecting software supply chains. It enables developers to cryptographically sign software artifacts, such as code, binaries, and container images, without needing to manage long-term keys. To prevent supply chain attacks, Sigstore offers an easy-to-use solution to ensure the authenticity and integrity of software artifacts throughout the development and deployment process. Sigstore achieves this through public verfiability: by logging all signatures in a tamper-evident, public transparency log, anyone can verify the provenance and integrity of software artifacts, fostering trust and transparency in open source ecosystems. This document provides a detailed overview of that transparency log, Rekor. 

### 1.1 Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 \[[RFC2119](https://www.rfc-editor.org/info/rfc2119)\] \[[RFC8174](https://www.rfc-editor.org/info/rfc8174)\] when, and only when, they appear in all capitals, as shown here.

## 2. Overview

The transparency service provides a tamper-evident, verifiable, append-only log to publicly record the existence of signing events, taking inspiration from Certificate Transparency ([RFC 6962](https://datatracker.ietf.org/doc/rfc6962)). Each record of the log contains the necessary metadata to verify the signing event without additional context or policy; generally, this metadata consists of the signature, the digest of the payload that was signed, and the key material or certificate used to sign. The log uses a Merkle tree structure where each entry is a leaf node. Merkle trees provide efficient and secure verification of the inclusion of a record in the log, and an efficient way to verify consistency, that the log remains append-only. The aim is to allow interested parties to audit signature issuance and validate existence of a signature at a particular instance of time, especially when used in conjunction with identity-based certificate issuance (see [Spec: Fulcio](https://github.com/sigstore/architecture-docs/blob/main/fulcio-spec.md)).

The transparency service implements the protocol operations for submissions and queries to the log defined in the Transparency Service API section below. It is responsible for verifying that submissions satisfy the *admission criteria* of the store: that is, the log only accepts structured data items that are self-contained valid records of a signature event. The log must also perform canonicalization to avoid duplication of equivalent formulations of the same signing event. For example, signing keys may be encoded by clients in either DER or PEM formats, but will be persisted in the log only in a PEM format.

The service MUST provide a mechanism for validating an entry’s inclusion and metadata (the integrated timestamp) that supports both online and offline verification. The log MAY use key material that commits to entry inclusion or entry metadata that is not included in the Merkle tree leaf values, which can support batching entry upload.

The service MAY act as a timestamping authority, by providing a signed inclusion time. If users want to distribute trust beyond the log operator, the transparency service SHOULD support user-provided signed timestamps that were generated during a signing event. Those signed timestamps will be incorporated into the leaves of the log to prevent tampering.

The service MUST NOT be relied upon for storage of signing events. Transparency systems act as witnesses to events. It is antithetical to the design of transparency systems to trust the contents of the log, since the log would be able to lie about its contents. Transparency systems are designed to be verifiable without needing any trust in the log.

A transparency service MUST be monitored by Witnesses that independently verify the integrity and append-only properties of the log, and Witnesses SHOULD distribute a signed commitment that the witness verified the log. Witnesses MUST be operated in a logically separate trust domain from the log operator. A transparency service MUST be monitored by individuals who can verify the contents of the log, for example artifact signers or key holders. These entities MUST verify that an entry in a log is expected.

The log service MAY also support an additional, unverified index that associates record properties with log entries to provide efficient queries. For example, interested parties may wish to retrieve signing events of a particular message. In this case, the log service may associate message digests with entry leaf values after submission into the index. This feature could transition to a verifiable index with the use of transparency maps, but this is currently not supported.

## 3. Architecture

The transparency service consists of the following major components:

1. An implementation of a Merkle tree server  
2. A Merkle tree “personality” that implements the logic for entry admission and the API surface  
3. A signer that provides signed responses to log inclusion and consistency proofs

## 4. Terminology

| Term | Definition |
| :---- | :---- |
| Merkle Tree | Cryptographic data structure in which every leaf node is represented by a cryptographic hash, and every parent node is the cryptographic hash of the concatenation of the children nodes. Allows for efficient proofs of inclusion and consistency. |
| Tree ID | Integer identifier for a log shard. Sometimes called a log ID. |
| Log ID | SHA-256 hash of the log's public key, calculated over the DER encoding of the key represented as SubjectPublicKeyInfo. Included as an identifier in a Signed Entry Timestamp. |
| UUID | 32-byte hex-encoded leaf hash of an entry. May be prepended with an 8-byte hex-encoded shard identifier called the Tree ID. |
| Entry ID | An EntryID is an artifact’s unique identifier and is made of two components, the TreeID and the UUID. Artifact lookup occurs by finding the UUID within the tree specified by the TreeID. |
| Log Index | Monotonically increasing value for each entry in a log shard. May also be referred to as a virtual log index. |
| Inclusion Proof | Proof that a given leaf value is committed to by a given Checkpoint. Consists of a list of ordered hashes of size log(N) where N is the size of the shard. |
| Consistency Proof | Proof that a given log is a prefix of another log, i.e. append-only. Consists of a list of ordered hashes. |
| Signed Entry Timestamp | Signature over the inclusion time, canonicalized entry, log ID and log index. Acts as a promise that Rekor will upload an entry.  |
| Checkpoint | Commitment by the log to its current state, containing the Merkle tree root hash and signed by Rekor's private key. A checkpoint also may be referred to as a Signed Log Root or Signed Tree Head. |
| Shard | An instance of the log. A new instance of the log is created on a regular interval to prevent the log from indefinitely growing, for scalability. The Transparency Service instance will have a unified interface over one or many shards. |
| Global Log Index | Log index computed across all log shards. |
| Witness | An entity which maintains its view of a "latest consistent checkpoint" from a log. May sign and expose the checkpoints it witnesses. Sometimes called a monitor. |

## 5. Records

The transparency service defines the structure of the admissible data in the log. Each entry stored in the Merkle tree must contain a verifiable record of a signing event and include the necessary information to validate the signature. The structure of the log record differs based on the type of the signing event and the signature format. The transparency service configures the admissible signing event types and formats based on the intended usage of the log.

### 5.1 Types

The transparency service defines structured typed entries that are admissible in the log. The log service may configure which types are accepted. At a minimum, an entry type should contain a message digest, signature or data structure containing it, and signature verifier such as a public key or certificate. The entry type may not be structured as a tuple, if the service is configured to understand the structured type. Another common entry may be a DSSE envelope, which contains a digest and signature, along with the provided verifiers. Entry types have type-specific validation and canonicalization, as described later in this document.

### 5.2 Signature Verification

When the service receives a proposed entry for submission, it must validate the signing event using the data contained in the record. In general, it will contain some representation of the data signed, the signature(s), and the key material used to verify the signature(s).

Validation metadata should include verification material which can verify the signature(s). Validation metadata will commonly be a code-signing certificate or a public key.

### 5.3 Canonicalization

The transparency service will perform type-specific canonicalization on records to de-duplicate equivalent entries. An equivalent entry is one that represents the same signing event. Sources of duplication include encoding schemes of the public key or signature. For example, the transparency service must de-duplicate an entry that encodes the public key with DER-encoding from one that uses the PEM-encoding. Canonicalization must not remove any necessary information to validate the signature. The service must store enough information in the Merkle tree leaf for an interested party to perform the same record validation to ensure entry auditability.

However, note that representing an entry with two distinct types will result in distinct entries, as canonicalization is only performed on entries of a certain type.

### 5.4 Example

```  
{  
    "apiVersion": "0.0.1",  
    "kind": "rekord"  
    "spec": {  
        "data": {  
            "hash": {  
                "algorithm": "sha256",  
                "value": "a4492b0eabdd2312f063290abd797d9d71a3ab28bd65a2c189b0df0d39b8c3b9"  
            }  
        },  
        "signature": {  
            "content": "MEQCICNdXy3bXp1DMOL6NPfX35gJ27bzldwSvCANwydOQUijAiBAh9ZRpCp3bX9xOTlHSGl4pUFwFmPRIXficOiE0G3Usw==",  
            "format": "x509",  
            "publicKey": {  
                "content": "\<Base64-encoded public key or certificate\>"  
            }  
        }  
    },  
}  
```

In this example, the entry type is denoted by `kind`. The type is also versioned with `apiVersion`. The tuple of a message digest, signature, and signature verifier is provided in `spec.data.hash.value`, `spec.signature.content`, and `spec.signature.publicKey.content` respectively. `spec.signature.format` specifies the signature verification type.

## 6. Privacy

A transparency service should consider the privacy implications of including sensitive data in the content of the entry. Contents from the log cannot be removed by design, since logs are append-only and any mutations would invalidate the log. Sensitive content could be included in the signature verifier, for example certificates could contain signer identities. Entities monitoring the log can also learn when specific artifacts are signed if they monitor for occurrences of the digest. This may be considered sensitive when combined with learning the identity of the signer from the entry.

When requesting an inclusion proof from the log, the transparency service will learn the entry that the signature verifier is interested in. Additionally, when requesting a consistency proof from the log, the service MAY learn the entry if there is a one-to-one mapping between checkpoint and entry, which can occur if the log produces a checkpoint with every entry uploaded. The transparency service SHOULD provide a checkpoint that maps to a number of entries, which will provide k-anonymity for requesters. Entry verifiers should take care when requesting online proofs, and should prefer offline proof verification when possible, as discussed below. There is also active [research](https://arxiv.org/pdf/2203.01661.pdf) to improve privacy for online verification, which has been a challenge for the certificate transparency ecosystem too.

## 7. Configuration

The transparency service shall support configuring which entry types are supported. Additionally, the service shall support configuring the supported signing key algorithms used to sign checkpoints and timestamps. The service shall support configuring hash algorithms for the digests of the entries.

## 8. Transparency Service API

### 8.1 Get Log Information

```  
GET /api/v1/log

Outputs:

* Current checkpoint  
* Log size for current shard  
* List of inactive shards 

GET /api/v1/log/publicKey

Inputs:

* Tree ID

Outputs:

* Public key for requested log shard

```

Note that the `publicKey` endpoint should not be used to fetch a log's trusted public key and is only informational. The log's public key MUST be distributed out of band.

### 8.2 Get Consistency Proof

```  
POST /api/v1/log/proof

Inputs:

* First size \- Size of the log where consistency has already been verified  
* Last size \- Size of the log where consistency needs to be proven  
* Tree ID \- Which log shard to verify consistency

Outputs:

* Consistency proof (as a list of ordered hashes)

```

### 8.3 Create Entry

``` 
POST /api/v1/log/entries

Inputs:

* Entry properties determined by the entry type, which will include either a message or its digest, a signature, and a signature verifier

Outputs:

* Canonicalized entry  
* Log ID  
* Log Index  
* Integrated time \- Time of inclusion in the log  
* Inclusion proof  
* Checkpoint  
* Signed entry timestamp

```

The service may synchronously add the entry to the log, and should return an inclusion proof and checkpoint. The checkpoint must be signed by the service and must be used to verify the returned inclusion proof. If the service chooses to asynchronously process entries, then the service must return a signed entry timestamp to convey a promise to include the entry in the log in a timely manner.

The canonicalized entry is returned to the caller so that the client does not have to be aware of canonicalization logic.

The client may compute the entry’s UUID by computing the digest of the canonicalized entry, since the canonicalized entry digest is the leaf hash.

### 8.4 Get Inclusion Proof by Index

```  
GET /api/v1/log/entries

Inputs:

* Global log index

Outputs:

* Canonicalized entry  
* Log ID  
* Log Index  
* Integrated time  
* Inclusion proof  
* Checkpoint  
* Signed entry timestamp

```

### 8.5 Get Inclusion Proof by Entry ID

```  
GET /api/v1/log/entries/{entryUUID}

Inputs:

* Entry UUID prepended with shard identifier

Outputs:

* Canonicalized entry  
* Log ID  
* Log Index  
* Integrated time  
* Inclusion proof  
* Checkpoint  
* Signed entry timestamp

```

### 8.6 Get Inclusion Proof by Search Query

```  
POST /api/v1/log/entries/retrieve

Inputs:

* List of global log indices  
* List of entry UUIDs  
* List of entries, optionally canonicalized

Outputs:

* List of entries and their metadata  
  * Canonicalized entry  
  * Log ID  
  * Log Index  
  * Integrated time  
  * Inclusion proof  
  * Checkpoint  
  * Signed entry timestamp

```

This API differs from others in that it accepts a non-canonicalized entry so that clients do not have to be aware of canonicalization logic in order to compute a leaf hash or entry UUID.

### 8.7 Searchable Index

```  
POST /api/v1/index/retrieve

Inputs:

* List of record properties

Outputs:

* List of entry UUIDs

```

The service may support an unverified index that associates record properties with log entries to provide efficient queries. The record properties may include the message digest, the subject of the certificate, or a fingerprint of a public key.

Since the index is not verifiable, the log can lie about inclusion or non-inclusion. Entry verifiers must not trust the index without also requesting an inclusion proof from the log using the returned entry UUID.

## 9. Verification

### 9.1. Merkle Tree

The log leverages a Merkle tree to provide efficient proofs of inclusion and consistency. The construction of the tree is as follows:

* For each entry in the log, create a leaf node whose value is the cryptographic hash of the entry  
* The values of parent nodes in the tree are the hash of the concatenation of the node's children. For example, with a binary Merkle tree, for each pair of leaf nodes, a parent node's value is the hash of the concatenation of the values of the children.  
* Continue creating parent nodes by concatenating children and taking the hash of the concatenation until there is only one node left.  
* The root will be referred to as the checkpoint, root hash or tree head.

An example of a Merkle tree:![][image1]  
Source: [https://en.wikipedia.org/wiki/Merkle\_tree\#/media/File:Hash\_Tree.svg](https://en.wikipedia.org/wiki/Merkle_tree#/media/File:Hash_Tree.svg)

### 9.2. Checkpoint Format

The transparency log must define a checkpoint format that includes:

* An origin string as a unique identifier for the log. It should be a URI.   
* Current log size  
* Current root hash of the tree  
* Optional additional metadata, such as a timestamp  
* A signature over all of the above

This checkpoint format is defined in [this specification](https://github.com/transparency-dev/formats/blob/main/log/README.md). The checkpoint format takes inspiration from [Golang's SumDB note format](https://pkg.go.dev/golang.org/x/mod/sumdb/note). An example of a checkpoint, which follows the above format:

```  
log.transparency.dev - 2605736670972794746
24582053
2VF265K09QM0UwRBoUOCC7c74/YWvmR6kiYPHy48Oqg=
Timestamp: 1690317786624821518

- log.transparency.dev wNI9ajBEAiAxjq6SjyzLZDicASq7m595Gb2tG7vrjNHWPTpPEE3SHAIgHb3XToE+0ShwkVSzw29eOcrp8ce4c5AW3YLcUNQsF+Q=  
```

### 9.3. Inclusion Proofs

Transparency systems provide proofs of inclusion where the log commits to a given leaf hash. Merkle trees provide an efficient design to compute and verify inclusion proofs. Inclusion proofs are composed of a list of hashes, a checkpoint, and the entry. To verify an inclusion proof, use the algorithm specified in [RFC 6962 2.1.1](https://datatracker.ietf.org/doc/html/rfc6962#section-2.1.1). The intuition for this algorithm is as follows:

* Cryptographic hash functions are assumed to have preimage, second preimage and collision resistance. Given an input, it is trivial to compute its digest, but given a digest, it is computationally difficult to compute its preimage.  
* A checkpoint acts as a root of trust for an inclusion proof since it's signed by the log.  
* It would be computationally difficult to find a preimage or collision that generates a given checkpoint, and a hash whose preimage would equal a forged entry.  
* Therefore, due to the construction of the Merkle tree, it is computationally difficult to create a list of hashes for a forged entry.

When verifying an inclusion proof, the verifier must use the entry log index and not the global log index. Similarly, the entry UUID, which is the hex-encoded log leaf hash, must not be prepended with the shard identifier.

A client must verify a checkpoint along with the inclusion proof. Otherwise, there is no signed commitment from the log to the root hash of the inclusion proof.

### 9.4. Consistency Proofs

Transparency systems do not inherently provide the properties of immutability and being append-only. Rather, they provide the ability to detect when mutations and removals occur. Merkle trees provide an efficient design to compute and verify consistency proofs, which demonstrates that a log is append-only between two log sizes. A consistency proof is composed of a list of hashes, a checkpoint for the initial log size, and a checkpoint for the current log size. To verify a consistency proof, use the algorithm specified in [RFC 6962 2.1.2](https://datatracker.ietf.org/doc/html/rfc6962#section-2.1.2). After a consistency proof is verified, a client should persist the latest log checkpoint, which will be used as the initial checkpoint when consistency is proved later.

### 9.5. Signed Entry Timestamp

The service may process entries asynchronously to handle bursts of traffic. In this case, the service will not be able to generate inclusion proofs synchronously. Instead, the service may return a promise of log inclusion, which is a signed commitment from the service to include the entry in the log in a given timeframe. The promise will be referred to as a Signed Entry Timestamp (SET).

An inclusion proof is always a stronger commitment than a promise. With only a promise, the client must request an inclusion proof from the log for verification of an entry, which requires an additional lookup. The transparency service should prefer synchronous processing of entries and returning an inclusion proof at the time of upload.

SETs also provide a signed timestamp from the log for when the entry is included. Timestamps are used within the Sigstore ecosystem to verify short-lived code-signing certificates (see [Spec: Fulcio](https://github.com/sigstore/architecture-docs/blob/main/fulcio-spec.md)). A signed timestamp should be over an entry signature, binding the current time to an observation of a signing event. The transparency service will provide a link from the signing key to the timestamp. A private key is used by the transparency service to sign a challenge when requesting a code-signing certificate and is used to sign an entry. By creating a timestamp over the entry’s signature, the transparency service provides an auditable proof of its possession of a signing key at a given time when the code-signing certificate was valid.

### 9.6. Offline Verification

[Spec: Transparency Service](https://docs.google.com/document/d/1NQUBSL9R64_vPxUEgVKGb0p81_7BVZ7PQuI078WFn-g/edit?pli=1#heading=h.qpdfgg3790iz) discusses the privacy consequences of online verification. Offline verification should be preferred when possible. Inclusion proofs and Signed Entry Timestamps are verifiable offline without requiring an online query to the log. Entry uploaders should distribute proofs alongside entries so that verifiers do not need to query the log.

Consistency proofs require querying the log. However, verification of a consistency proof once one is obtained can be done offline. By including a consistency proof along with an inclusion proof, a verifier will know that an entry is in the log and the log was consistent at the point of inclusion proof generation.

## 10. Threats

### 10.1. Consistency

As discussed above, transparency services are not inherently immutable, but provide an efficient method to verify that the log is consistent and remains append-only between two states of the log. To verify consistency, a client should query the log to request a consistency proof between a trusted state and the current state. This puts a significant burden on the client to make verification an online process and require that the client keep state by persisting checkpoints.

A witness serves the role of verifying that the log remains append-only by periodically requesting consistency proofs. A witness will persist a trusted checkpoint, periodically request and verify a consistency proof to the latest checkpoint, and then persist the latest checkpoint.

### 10.2. Split View

A malicious or misconfigured transparency service may present a different view of the log to different clients, called a "split-view attack". For example, consider a situation where a client A uploads an entry E to the service. The service maintains two logs, log X and X'. X will contain the entry, X' will not. Client A requests an inclusion proof for E from the service to confirm the entry is uploaded, and the service returns a proof based on X. Client B requests an inclusion proof for E, and the service returns no proof, effectively causing a denial of service. Furthermore, the log could lie about its contents and present an inclusion proof even if no entry had been uploaded.

To mitigate this attack, there must exist more than one a witness that verifies consistency. If there is only one witness verifying consistency, then the log could present a view to the witness that differs from the view it presents to other clients. If the witness were addressable, the client could query the witness to learn what its most recent checkpoint is, and then request an inclusion proof from the log for this checkpoint. However, the witness and transparency service could collude to lie to the client, or the service could present a split view to the witness without the witness knowing.

One witness is not sufficient to mitigate the attack. There must exist multiple witnesses, or a network of witnesses, that verifies consistency and distributes their verified checkpoints. This process is referred to as gossiping in [RFC 6962](https://datatracker.ietf.org/doc/html/rfc6962), where witnesses share their verified checkpoints with other witnesses. Instead of requiring that witnesses gossip with each other, witnesses can distribute their checkpoints to a set of "distributors". This makes witnesses lightweight, responsible only for verifying checkpoints and persisting state, while the distributors are responsible for aggregating checkpoints and verifying no split view occurs. Distributors also will be addressable so that clients can query to get the latest, verified view.

If the service presents a split view to any witness, that witness will distribute its checkpoint to a distributor who is receiving checkpoints from other witnesses. If the distributor sees that a witness has received a checkpoint whose log size matches other distributed checkpoints but whose root hash differs, then the distributor knows a split view has occurred and must alert the ecosystem.

#### 10.2.1. Stable Checkpoint

The transparency service may create a new checkpoint for every entry uploaded. If this is very frequent, then the checkpoint will not remain stable long enough for two witnesses to verify the same checkpoint. While distributing their checkpoints, it will not be possible to reach a consensus on the same checkpoint, meaning that split-view attacks cannot be mitigated.

If the service frequently updates its checkpoint, then the service must publish a "stable checkpoint" periodically. As long as the period is long enough that multiple witnesses can verify consistency during that period, then a distributor can verify that no split view is occurring.

A consequence of this approach is that clients who want to verify that no split view is occurring must wait until multiple witnesses have verified consistency from a stable checkpoint. Therefore, a preferable approach would be for entry verifiers to only verify an inclusion proof and the latest checkpoint, and require another set of entities in the ecosystem to be responsible for detecting split-view attacks. There must be some set of entities in an ecosystem that monitor for split-view attacks, but it's not a requirement that every consumer do the same calculation. This is an area of active research.

### 10.3. Entry Key Compromise

The transparency service records signing events, which includes a signature verifier such as a certificate or public key. If a key or certificate were to be compromised, a malicious actor could use compromised material to generate valid signatures over tampered artifacts. Artifact verifiers must verify that an artifact is present in the log in order to trust the artifact. If the attacker does not upload the artifact and its key to the log, then the artifact must not be trusted.

Transparency services force malicious actors into the open. A malicious actor could use a stolen key to generate a valid signature over a tampered artifact and upload it to the log. However, this now forces the attacker into the open. The owner of the key must monitor the transparency log for occurrences of their key. If the owner sees unexpected occurrences, then the owner knows that their key is compromised and must alert those who trust their signed artifacts. 

## 11. Pluggable Types

The transparency service supports uploading entries with a range of formats, defined as a "pluggable type" system. A pluggable type is a custom schema for entries stored in the transparency log.

Current supported types are:

* Alpine packages ([v1](https://github.com/sigstore/rekor/blob/main/pkg/types/alpine/v0.0.1/alpine_v0_0_1_schema.json))  
* COSE envelopes ([v1](https://github.com/sigstore/rekor/blob/main/pkg/types/cose/v0.0.1/cose_v0_0_1_schema.json))  
* DSSE envelopes ([v1](https://github.com/sigstore/rekor/blob/main/pkg/types/dsse/v0.0.1/dsse_v0_0_1_schema.json))  
* HashedRekord ([v1](https://github.com/sigstore/rekor/blob/main/pkg/types/hashedrekord/v0.0.1/hashedrekord_v0_0_1_schema.json))  
* Helm provenance ([v1](https://github.com/sigstore/rekor/blob/main/pkg/types/helm/v0.0.1/helm_v0_0_1_schema.json))  
* in-toto attestations ([v1](https://github.com/sigstore/rekor/blob/main/pkg/types/intoto/v0.0.1/intoto_v0_0_1_schema.json), [v2](https://github.com/sigstore/rekor/blob/main/pkg/types/intoto/v0.0.2/intoto_v0_0_2_schema.json))  
* JAR / Java archives ([v1](https://github.com/sigstore/rekor/blob/main/pkg/types/jar/v0.0.1/jar_v0_0_1_schema.json))  
* Rekord ([v1](https://github.com/sigstore/rekor/blob/main/pkg/types/rekord/v0.0.1/rekord_v0_0_1_schema.json))  
* RFC 3161 timestamps ([v1](https://github.com/sigstore/rekor/blob/main/pkg/types/rfc3161/v0.0.1/rfc3161_v0_0_1_schema.json))  
* RPM packages ([v1](https://github.com/sigstore/rekor/blob/main/pkg/types/rpm/v0.0.1/rpm_v0_0_1_schema.json))  
* TUF metadata ([v1](https://github.com/sigstore/rekor/blob/main/pkg/types/tuf/v0.0.1/tuf_v0_0_1_schema.json))

Each schema specifies how the transparency service should parse a request to get an entry or its digest, an entry's signature, and the signature verifier. The recommended types to use are HashedRekord and DSSE for uploading signed artifacts and attestations respectively.

HashedRekord and Rekord provide basic schemas for submitting an entry, signature, and signature verifier. HashedRekord requires the entry digest, and Rekord requires the entry. The recommendation is to use HashedRekord. The type is better for performance since the entire artifact does not need to be sent to the service. Additionally, the type is more privacy conscious since the client does not need to submit the entry to the log. However, note that the log only records the digest of the entry for all pluggable types even if the schema requires the entry on upload.

in-toto has been superseded by the DSSE type. Unlike the in-toto type, the DSSE type does not require storage of attestations alongside the log. in-toto attestations must be stored outside of the transparency service, as the service must not be used as storage. Transparency logs are designed to be trustless, so it is an antipattern to rely on the log to provide metadata, which is why it is recommended to use the DSSE type.

## 12. Pluggable Verifiers

Each entry uploaded to the transparency service requires a signature and a signature verifier. The service's admissions policy is to only admit entries with a valid signature. The service supports a "pluggable verifier" interface that defines cryptographic verifiers in a variety of formats.

Current supported verifiers include:

* X.509 certificates  
* RSA, ECDSA, and Ed25519 PEM-encoded PKIX public keys  
* SSH keys  
* [Minisign](https://jedisct1.github.io/minisign/) keys  
* TUF root metadata  
* PGP keys  
* PKCS \#7 encoded public keys

Not all verifiers are supported for each pluggable type. For example, HashedRekord only supports X.509 certificates and PKIX public keys.

## 13. Sharding

Creating new log trees, or "sharding", handles the issue of an indefinitely growing log. Smaller logs are more efficient to query. A large log is difficult to maintain for many reasons - Turning up a new witness, search index, or mirror will take a significant amount of time, and storage can become costly. Additionally, logs should periodically rotate their signing keys for good key hygiene, and periodic sharding becomes a natural point to do this rotation.

[RFC 6962](https://datatracker.ietf.org/doc/html/rfc6962) certificate transparency logs handle sharding by creating a new shard every year. For CT, each shard is accessible on its own URL, logid.domain.com/year. Clients and witnesses need to update to the latest log shard URL each year.

This transparency service has implemented sharding to avoid any yearly updates for clients. Entries will always be accessible via the same URL. Each year, the log operator will create a new tree, and update the service to write to this new tree by default. There must be only one active tree. All other trees in the service will be frozen and non-writable.

Old entries will still be accessible by entry ID, which is a UUID prefixed with the log ID where the entry resides. Additionally, entries can be accessed via their global log index, which informs the service's lookup across shards.

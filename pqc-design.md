# Post-Quantum Cryptography for Sigstore

This document is a read-only copy of the design document on
[Google Docs](https://docs.google.com/document/d/1g3WQk4JSwQ26MGwB-Kkhu8ylrPF5JnE4v62d2tDC2xY/edit?tab=t.0#heading=h.nggvg4888mfa)
(join [sigstore-dev](https://groups.google.com/g/sigstore-dev) to view, do not request access).
We will make a best effort to keep it in sync with the original document, though the source of truth will
be the original document for the time being.

## Executive Summary

To meet expedited industry timelines targeting 2029 for quantum-resistant cryptography, Sigstore is introducing immediate support for the post-quantum cryptographic (PQC) signature scheme ML-DSA (FIPS 204). Because PQC introduces significantly larger signature and public key sizes, this transition serves as a catalyst for a major architectural simplification of Sigstore's infrastructure.

This document outlines a revised design that streamlines the signing and verification processes. Key proposals include:

* **Strong Security Guarantees:** Achieving these architectural simplifications without altering or weakening the existing Sigstore security and threat model.  
* **Simplified "Keyless" Signing:** Eliminating the need for short-lived X.509 code signing certificates and signed timestamps in favor of direct identity token verification in the transparency log.  
* **New Log Entry Format:** Hashing all user-provided data and metadata (including large PQC keys and signatures) in the log leaf to mitigate log poisoning, reduce storage and egress costs, and address privacy concerns, while ensuring transparency log proofs continue to provide artifact provenance.  
* **Strict Backward Compatibility:** Deploying an independent infrastructure stack for the public-good infrastructure, and establishing separate client APIs to ensure existing users are not broken.  
* **Flexible Signing Models:** Continuing to support additional signing models, such as managed keys and traditional PKIs, to accommodate offline signing scenarios and private deployment use cases.  
* **Standards Alignment:** Adopting proposed C2SP specifications for transparent signature formats and identity-based transparency logging.

Ultimately, this design provides a resilient, scalable, and simplified path forward for post-quantum artifact signing.

## Background

Post-quantum cryptography (PQC) is an emerging necessity for applications that deal with encryption and signing. Quantum confidentiality, where data that is encrypted now can be decrypted later (e.g. harvest-now-decrypt-later attacks on TLS), is a critical use case, and there is active work for the web and organizations to adopt quantum-resistant encryption key exchange algorithms (e.g. ML-KEM). This threat doesn't affect digital signing applications though. Breaking quantum integrity/authenticity would allow the forging of a signature, but ecosystems can mitigate this threat by rejecting classical cryptographic algorithms at the time of signature verification. However, artifact signatures can be valid indefinitely, and simply re-signing artifacts is not always possible.

Google has [expedited its timeline](https://blog.google/innovation-and-ai/technology/safety-security/cryptography-migration-timeline/) to 2029 for a migration to quantum-resistant cryptographic algorithms due to [significant improvements](https://research.google/blog/safeguarding-cryptocurrency-by-disclosing-quantum-vulnerabilities-responsibly/) in the resources needed to build cryptographically relevant quantum computers (CRQCs), with Cloudflare [announcing](https://blog.cloudflare.com/post-quantum-roadmap/) the same timeline. Sigstore had planned to follow NIST's timeline, aiming to integrate PQC into Sigstore by 2030 at the latest with experimental infrastructure and tooling between now and then.

Given these expedited timelines, Sigstore will move to support PQC algorithms immediately. This document describes the changes to infrastructure and clients to support PQC. The proposed changes are a significant revision to Sigstore, not only to support PQC but also to greatly simplify signing and verification while maintaining the same security & threat model.

This design document is primarily focused on solving the needs of the public-good infrastructure and broader open source communities. At the end of this document, we include a discussion on private deployments, and we welcome feedback on the requirements for private deployments.

## High-Level Requirements

To support post-quantum cryptography for Sigstore, which we'll refer to as "PQC Sigstore", we will meet the following high-level requirements.

### Do not break existing users

Any changes will not affect existing users in any way. Users will opt in to PQC support. Concretely, this will mean an independent infrastructure stack and new signing and verification APIs in client SDKs.

### Do not break users again

When we began to roll out Rekor v2, we acknowledged that introducing this change would break verifiers, where the signer logs to Rekor v2 but the verifier doesn't understand how to verify the leaf hash. This introduced significant complexity with the rollout, as Sigstore is adopted by public package registries that cannot control package manager versions.

Even if we do not break existing users, we recognize that adopting PQC is a breaking change for an ecosystem. We do not want to introduce multiple breaking changes in a short timeframe.

However, we also acknowledge that this is a unique opportunity to act on all of the learnings from designing and operating Sigstore over the last 5 years. From the user's perspective, one breaking change isn't different from multiple breaking changes.

While we cannot guarantee yet another revision of Sigstore in the distant future, we will address all present concerns while supporting PQC to minimize the likelihood of another large breaking change. It is this new design that would be taken to a standards body as well.

#### Reduce complexity where possible

To sign with Sigstore, a client must interact with many components: An identity provider, certificate authority, timestamp authority, and transparency log. Similarly, to verify a signature, a client must verify a signed timestamp, a certificate's signature, subject and validity, an artifact's signature with the certificate's public key, and a log proof. While each step has a purpose, it is a complex architecture and this has led to occasional bugs with skipped steps. As mentioned [below](#infrastructure-must-scale-with-large-signatures-and-public-keys), this amount of signature metadata also affects log storage and bundle sizes.

We have an opportunity to revisit the client specification and determine where we can reduce complexity while maintaining the same security posture and threat model.

#### Handle log poisoning and privacy concerns

We are aware of the existential threat of log poisoning, where users maliciously persist spam or problematic content in the append-only transparency log to force the log to be taken down, as an ecosystem-wide denial of service.

We also would like to address privacy concerns that have been raised in the past with the logging of emails, workload identifiers, and private repositories. While we will not prevent linkability between logged signatures, we can redact sensitive data.

PQC Sigstore will address both of these concerns with a new leaf type that contains only hashes of user-provided content.

### Do not commingle classical and post-quantum cryptography in API Design

As we design new client APIs and offer new infrastructure, we should not intermingle classical cryptography and post-quantum cryptography. We want to avoid building single APIs that can accept signatures that offer different security strengths. Additionally, keeping a clean separation between the two modes provides a straightforward way for the eventual deprecation of older APIs and infrastructure.

Note that this API recommendation is not for applications or tools that build on top of Sigstore, which include first-party applications like Cosign or policy-controller, and numerous other tools which process signed provenance. Based on ecosystem needs, applications can make their own choices for how to enable a transition between the two. For example, an application could initially accept both classical and post-quantum signatures, accepting the tradeoff between consumer usability and acceptance of any signing algorithm. Over time, the application can change its default policy to only accept one or the other, again accepting the tradeoff that this breaks verification for some artifacts but provides stronger security guarantees.

For Cosign, we will support all signature algorithms initially, with a flag to opt-in to exclusively accepting PQC signatures. In a future major revision, well before any required migration timelines, Cosign will accept only PQC signatures, with a flag to opt-out and accept all. In a later major revision, PQC signing algorithms will only be accepted.

[Enabling a Sigstore PQC Transition](https://docs.google.com/document/d/1iHYdz3klOQUv8sDzVwwbMdE262n7xFGdKLqX92xhkig/edit?resourcekey=0-wFqCGMFcoIJXywavzKSncg&tab=t.0) provides additional discussion on this topic for application developers.

### Align with industry standards

Sigstore has always aimed to follow the industry and leverage open standards rather than reinvent the wheel. Sigstore will adopt the PQC algorithms that are recommended by cryptographers, which at this time is ML-DSA. We will not support HashML-DSA as [recommended](https://keymaterial.net/2024/11/05/hashml-dsa-considered-harmful/), and we will aim to not support external-μ, instead recommending applications devise their own pre-hashing schemes.

### Infrastructure must scale with large signatures and public keys {#infrastructure-must-scale-with-large-signatures-and-public-keys}

ML-DSA signatures and public keys are significantly larger than ECDSA & Ed25519.

* Signature size (in bytes)  
  * ML-DSA-44: 2420  
  * ML-DSA-65: 3309  
  * ML-DSA-87: 4627  
  * ECDSA P-256: 64  
  * Ed25519: 64  
* Public key size (in bytes)  
  * ML-DSA-44: 1312  
  * ML-DSA-65: 1952  
  * ML-DSA-87: 2592  
  * ECDSA P-256: 65  
  * Ed25519: 32

At scale, this significantly increases storage and egress costs. To minimize those costs, we will redesign the log leaf while maintaining the same threat model.

### Signature bundle size is reasonable

While the signature bundle size is likely going to be much smaller than the artifact, we should still strive to keep signature bundle sizes reasonable, as large metadata is a concern that has been raised by package registries in the past.

## Security & Threat Modeling of Current Protocol

Before discussing any changes to Sigstore, we will articulate a security model, defining what Sigstore should achieve, and a threat model, describing what threats Sigstore mitigates and how, for the current Sigstore protocol and infrastructure.

### Security Model

| Security Property | How It's Achieved |
| ----- | ----- |
| Sigstore provides integrity for artifacts | Cryptographic signatures |
| Sigstore provides authenticity for artifacts | Signatures bound to user or workload identities, or user-controlled keys |
| Users do not have to manage signing keys | Users request a short-lived code signing certificate for an ephemeral signing key, authenticating with an OpenID Connect identity token |
| Short-lived certificates are trusted long-term | Timestamp authority issues signed timestamp binding time to an artifact signature |
| Signatures are auditable | Append-only transparency log records a key/certificate, artifact hash, and signature |
| User identity verification is auditable | Certificate authority transparency log records issued certificates |
| Infrastructure is widely trusted | Trust root material distributed via TUF, signed by a quorum of offline keyholders from academia and trusted organizations |
| Users can discover malicious signatures or compromised identities | Transparency logs are public and monitored by signers |

### Threat Model

| Threat | Mitigation |
| ----- | ----- |
| OpenID Connect provider is compromised, identity token misissued | When the identity token is provided to the certificate authority, the CA records the identity-bound certificate to a transparency log |
| User credentials compromised | Transparency log records any usage of the credential for certificates or signatures. Additionally we rely on the security of the identity provider (e.g. mandating 2FA) |
| Certificate authority compromised | All issued certificates are recorded to a transparency log |
| Compromised identity used to sign an artifact | Artifact signing is recorded to an auditable transparency log. Signature verifiers require an inclusion proof, meaning that the signature is publicly auditable. With no proof or an invalid proof, they do not trust the signature. |
| Transparency log (certificate or signature log) compromised, hiding maliciously generated signatures in distinct log views for different callers | Transparency log must be witnessed\[1\] to prevent split views, with independent witnesses verifying the consistency (append-only property) of the log for the same log checkpoint. |
| Transparency log witness is compromised | Multiple witnesses co-sign a checkpoint |
| A quorum of witnesses is compromised | A significant quorum such that this is infeasible |
| Public instance root key holder compromised | Quorum of keyholders required to make changes |

\[1\] \- Rekor and the certificate transparency log are not currently witnessed due to a lack of specification support (RFC6962 doesn't support witnessing) and too small of a batch size (Rekor issues a new log checkpoint with every entry, so witnesses cannot easily co-sign the same checkpoint). The Tessera-based Rekor and TesseraCT will be witnessed.

## Reducing Signatures & Complexity

### Current Verification Metadata

A Sigstore signature generated with an identity token as part of a "keyless" signing flow produces the following metadata:

* 1 artifact signature  
* 1 ephemeral public key, included in the code signing certificate  
* 1 CA signature for the code signing certificate  
* 1 SCT promise, a commitment of inclusion from a certificate transparency log  
* 1 timestamp signature, issued by a TSA over the artifact signature  
* 1 log checkpoint signature, binding an inclusion proof to a log  
* N witness signatures, independently verifying log consistency

We don't include trusted root material, which must be distributed out of band.

Note that a Sigstore signature can also be generated with a managed key. The verification materials would then not include a certificate or timestamp:

* 1 artifact signature  
* 1 log checkpoint signature  
* N witness signatures

Note that the public key is not included here, as that is expected to be distributed out of band as trusted root material.

### Truly Keyless Signing

tl;dr \- What if the log directly accepted identity tokens?

Let's first consider the purpose of a certificate. A certificate binds a subject (for Sigstore, an OpenID Connect identity) and public key (by default, an ephemeral key). A certificate is issued by a trusted certificate authority, whose issuance is auditable with a certificate transparency log.

To upload to the log, the log requires an identity to associate an artifact hash with, and a "credential" for that identity. In the current design, an identity and credential can be:

* A public key and a signature. This provides non-repudiation, non-refutable evidence that a signer who possesses a public key signed an artifact.  
* A certificate, which contains an identity. The certificate authority is trusted to have verified the original credential which authenticated the identity (an OpenID Connect identity token).  
  * Note that the artifact signature is by default *not* a credential. The default Sigstore signing flow generates an ephemeral signing key with the certificate binding that key and an identity. The key is not long-lived, and so the key, and by extension the signature, is not a proof of authenticity. That signature only provides integrity for an artifact, but the root of trust for that signature is the CA, not the key.  
  * The log also accepts certificates issued from non-Sigstore PKIs where signing keys may be a valid identity, but the log has no way to distinguish between managed and ephemeral keys.

The log records this identity and credential. Note that these credentials are made public in the log leaf, which is fine because the secret (the private key, ephemeral or not) remains private.

Let's consider another credential: The identity token. The identity token of course contains an identity. The identity token also represents a credential for that identity, issued by a trusted OpenID Connect provider. What if that token were provided directly to the log?

* The log would be responsible for verifying the token's signature.  
* The log would extract relevant information from the token (e.g. subject, issuer, provenance claims). Instead of encoding that information in a certificate, the contents would be a part of the leaf.  
* Note that unlike an artifact or certificate signature, the token signature must not be made publicly available in the leaf. The token signature is a secret credential, and anyone who possesses it would be able to forge an association between an artifact hash and an identity.

Without a certificate, how do we associate an ephemeral signing key with a token? We could associate a signing key with an identity token with a technique similar to [Demonstrating Proof of Possession](https://datatracker.ietf.org/doc/html/rfc9449). However, since this key is ephemeral, its signature still does not represent a meaningful credential.

What if we had no ephemeral signing key? Do we still have integrity and authenticity for an artifact? An artifact instead can be verified only by an inclusion proof.

* The log leaf hash is computed using the artifact hash and the claims from the identity token. Tampering with the artifact or claim values would lead to a different log leaf hash, causing proof verification to fail.  
* The proof is bound to a checkpoint, and the checkpoint is signed by the log and witnesses. Tampering with the proof would also lead to a verification failure.  
* The signer identity is required to be known out-of-band. Providing a different identity for the log leaf computation would also cause proof verification to fail.

Would this change affect Sigstore's threat model?

* The log, rather than the CA, is trusted to have verified the identity token. If the log were to forge an entry, it would be auditable. This is the same as with the CA recording issued certificates to a transparency log.  
* For the public instance, the log and the CA are operated by the same entity in the same trust boundary. Trusting the log or trusting the CA to verify an identity token is trusting the same operator.  
* An artifact signer could submit to multiple independent logs to avoid trusting a single log operator. This replicates Web PKI's usage of certificate transparency, where a certificate is logged to multiple logs.

This simplified design makes it more apparent that the Sigstore infrastructure is trusted to have verified an identity token, but this is no different from the current design. Identity token verification is an auditable process. Proving that the CA or log received a real identity token can improve both the current design or this new design, but this is orthogonal to reducing complexity. We will discuss a proposal for a zero-knowledge proof of a valid signature [later](#non-repudiation-&-credential-verifiability).

### Removing Signed Timestamps

Since there are no longer certificates in PQC Sigstore, we have no need for signed timestamps.

Timestamps served two purpose:

* To elongate the validity of a short-lived code signing certificate. Arguably, this was only necessary because X.509 certificates include a timestamp. A signature *should* be valid indefinitely.  
* To demonstrate possession of a signing key at the time of signature generation.

To the latter point, timestamps can be used when creating verification and revocation policies. For example, if I know my signing key or identity was compromised from time X to Y, then I can build a verification policy for a given artifact that permits signatures from a given identity from time \[0, X) and \[Y, current\] and disallow signatures from time \[X, Y).

In this new design, verification policies can still be built with timestamps, though log indices are another option.

* Witnesses generate cosignatures over a timestamp at about a 5 second interval, and so checkpoints are timestamped. Witnesses provide an independent view of time.  
* Log indices can provide a validity period as well, for example disallowing signatures from the first known malicious entry to when a valid entry was committed to the log.

### New Verification Metadata

Summarizing, we go from

* 1 artifact signature, 1 ephemeral public key, 1 CA signature, 1 SCT promise, 1 timestamp signature, 1 log checkpoint signature, N witness signatures, to:  
* 1 log checkpoint signature, N witness signatures for identity-based signatures, with 1 additional artifact signature for managed keys

For [private deployments](#private-deployments) that operate a certificate authority rather than a log, there will be no change, with 1 artifact signature, 1 ephemeral public key, 1 CA signature, and 1 timestamp signature.

## High-Level Design

To support PQC in Rekor and minimize the log leaf, we will implement an [identity-based transparency log spec](https://github.com/C2SP/C2SP/pull/244). This specification proposes a new leaf type where all user-provided content is hashed. Signatures, public keys, and identity token key/values are hashed, which concurrently mitigates log poisoning, privacy concerns and size constraints.

Rekor will accept identity tokens and public keys/signatures as credentials.

We will build on sigstore/rekor-tiles with a v3 module.

We will design a solution for private PKIs alongside the initial MVP (discussed more [below](#private-deployments)), based on customer needs. This will include support for certificates and signed timestamps issued from a separate set of services. Rekor will not include support for certificates as credentials since we do not have a known customer need where a public key or identity token cannot directly be used as the credential.

Sigstore clients will support ML-DSA and the proposed simplified signing and verification flow. The client specification and conformance suite will be updated. A Sigstore client may choose to implement a new SDK, or integrate support into the existing SDK as long as the APIs between the two flows cannot be intermingled.

A public-good instance will be deployed as an independent infrastructure stack. A new TUF root will be issued signed with ML-DSA root keys. A client will opt into this infrastructure stack by configuring their client to use the PQC TUF repository, which will include the URLs for the new infrastructure distributed via a signing configuration file and ML-DSA public keys for logs and witnesses distributed via a trust root file.

## Tradeoffs

These architecture changes present a number of tradeoffs. Some of these tradeoffs are necessary for the long-term strategy of the project, but they can have an impact on private deployments or existing tools. Comparing the current architecture and metadata with the PQC proposal, we highlight the following changes:

| Current | New | Motivation | Downside | Why It's Acceptable |
| ----- | ----- | ----- | ----- | ----- |
| Rekor stores complete certificates. | All user data is hashed in a new leaf format. | Fully mitigates log poisoning, where no user-provided content will be present in the leaf. PQC keys and signatures are much larger than ECC, and so hashing keys and signatures will reduce storage costs. | Content can no longer be discovered that's unknown, e.g. a monitor cannot match entries against regular expressions. | Transparency provides a way for an entity to verify a log claim. Discovery of the unknown risks noisy monitoring as a monitor may pick up entries that are not relevant. Monitors can build complete lists of what they know to look for. Provenance values are still included in the leaf, just hashed. Where there are known substrings to match against that are not already included, such as domains in an email, an additional context value can be added to the leaf. Log poisoning is an existential risk to a public log. Regardless of PQC, hashing was a necessary change. |
| Certificates contain provenance data | For log-backed deployments, log proof structure ([c2sp.org/tlog-proof](http://c2sp.org/tlog-proof)) contains provenance data. For private deployments that don't want to use a log, certificates can still contain provenance data. | Moving away from certificates by default reduces the number of signatures and for overall simplification while maintaining security properties. | Verifiers would have to understand both formats, since private deployments may still use certificates. Project-specific parser needed for the provenance data in the log proof. | Applications that verify Sigstore-signed provenance already have to use Sigstore libraries, so it should be straightforward to also provide APIs for parsing. X.509 is a heavyweight format, and adopting a simpler format may help adoption for lower level applications. Off-the-shelf X.509 parser libraries weren't guaranteed to work regardless. Some couldn't handle custom OIDs, and clients already had to be aware of multiple encodings for provenance, either as a raw string or ASN.1 encoded string. If certificates are a requirement, the tooling will support privately deployed CAs. |
| Timestamping with RFC3161 | Timestamping with witness cosignatures | As log proofs now act as a signature, witnessing provides the third-party assertion on the time of generation. | Verifiers will have to understand multiple formats for timestamps, since private deployments may still use RFC3161 TSAs. | Third-party timestamps are still available through witnesses. Timestamping is no longer a strict requirement, without short-lived certificates. Timestamps could be used to define admission policies, e.g. distrusting artifacts signed between time X and Y due to a compromise. An alternative is to use log index ranges instead. |
| Identities embedded in certificates are logged to both a CT log at time of issuance and an identity log at time of usage. | An identity bound to an artifact is logged at time of usage. | For transparent signatures with an identity token as a credential, certificates as an intermediate credential are not needed to offer the same security guarantees. | Private deployments will need to choose between a log-based model or a CA-based model, with the latter not recording certificates to a log. | Having two logs introduced a failure mode \- How should a monitor handle where one log is missing an expected entry? No tooling handled this case, and so the proposed simplification simply removes this as a possibility. Removing one log reduces deployment costs for public infrastructure. Private deployments that have determined a need for transparency will also have reduced costs. However, private deployments should determine whether a transparency log is even needed by authoring a threat model, or if a simpler alternative for an auditing mechanism would suffice. Proposals around PQC certificate transparency have been designed for Web PKI use cases, and the added complexity should be avoided for Sigstore. |

## Detailed Design

This design will focus on the infrastructure changes and changes to Go clients (sigstore-go, Cosign). Additional clients are encouraged to create their own design documents. Each client is expected to use a well-tested and vetted ML-DSA implementation.

### sigstore/sigstore

We will add support for ML-DSA in the [`signature` package](https://github.com/sigstore/sigstore/tree/main/pkg/signature) using [filippo.io/mldsa](https://pkg.go.dev/filippo.io/mldsa), which implements the [proposed crypto/mldsa API](https://go.dev/issue/77626) and exports the `crypto/internal/fips140/mldsa` package. `crypto/mldsa` is expected to land in Go 1.27, at which point we will switch to directly calling the upstream API.

Note that until support lands upstream, we will have to add special handling for PKIX marshaling as defined in [RFC9881](https://datatracker.ietf.org/doc/html/rfc9881). We will update functions like [`UnmarshalPEMToPublicKey`](https://github.com/sigstore/sigstore/blob/311895e7870187320e47337734a9c321c0a8819c/pkg/cryptoutils/publickey.go#L50) to conditionally call either `x509.ParsePKIXPublicKey` or an internal package for parsing ML-DSA keys. Upstream support [will land in Go 1.27](https://github.com/golang/go/issues/78888).

We will need a utility to parse ML-DSA signed JWTs for OpenID Connect tokens, following the [ML-DSA JOSE specification](https://datatracker.ietf.org/doc/rfc9964/). go-jose [plans](https://github.com/go-jose/go-jose/issues/194) to implement support eventually, at which point we can remove any custom utilities.

We will temporarily update the [certificate utilities](https://github.com/sigstore/sigstore/blob/main/pkg/cryptoutils/certificate.go) to parse ML-DSA signed X.509 certificates to support [private deployments](#private-deployments), though depending on timing, we may simply wait until Go 1.27 lands with support for RFC9881 identifiers in `crypto/x509` to avoid maintaining a temporary fork.

### Rekor

We will create a new Rekor v3 module in sigstore/rekor-tiles that supports a single log leaf format as defined in the [identity-based transparency log spec](https://github.com/C2SP/C2SP/pull/244). Concurrently, we would like to standardize this specification in C2SP, and are collaborating with Sigsum on a unified leaf format. Leave any comments on the overall specification on the GitHub issue. We summarize the proposal and include details on the Sigstore-specific leaf.

To mitigate log poisoning and minimize log leaf size, the log leaf will be composed of a list of hashes:

```
0x01 ||
Hash(root of trust) ||
Hash(message) ||
Hash(Hash(key1)) || Hash(Hash(value1)) ||
Hash(Hash(key2)) || Hash(Hash(value2)) ||
... ||
Hash(Hash(keyN)) || Hash(Hash(valueN)) ||
H(receipt)
```

`message` is a pre-hashing scheme for some data to sign, `root of trust` is the credential issuer (e.g. OIDC identity issuer, public key), the `key`/`value` pairs are ecosystem-specific strings, and the `receipt` is unrefutable proof of a credential.

For Sigstore, the ecosystem-specific strings will be OIDs mapped to values, mirroring Fulcio certificate extensions. For provenance values, the log will standardize token claims across CI providers exactly as Fulcio does.

We will also include a hash of the credential as a receipt. For a public key, this credential will be a signature. For an identity token, there is no credential hash because the credential is short-lived and not stored with the artifact, and so the receipt will be 32 `0x00` bytes. We discuss [below](#non-repudiation-&-credential-verifiability) an alternative using a zero-knowledge proof of a credential.

#### Leaf hash spec

`root of trust` MUST be either:

* Issuer of the OIDC token, as a string. Value may come from `iss` JWT claim or from a specified claim populated by a federated issuer like Dex.  
* Public key, PKIX ASN.1 DER-encoded in [SubjectPublicKeyInfo format](https://rfc-editor.org/rfc/rfc5280.html#section-4.1), prefixed with an algorithm identifier (`"ML-DSA-44"`, `"ML-DSA-65"`, or `"ML-DSA-87"`)

All JWT claim values will be extracted as-is without normalization. Monitors will be responsible for handling deviations, e.g. `"example.com"` vs `"Example.com"`. Provenance claims from OIDC CI tokens will be mapped to a standardized set of claims as per the [current Fulcio specification](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md#mapping-oidc-token-claims-to-fulcio-oids). All claim values will be treated as strings. Numbers will be converted to strings, and boolean values will be converted to `"1"` and `"0"`.

Key/value pairs will be sorted lexicographically by the digest of the key.

#### API

The API will accept the following credentials:

* Public key & signature, for ML-DSA-44, ML-DSA-65 and ML-DSA-87  
* OpenID Connect identity tokens  
  * Initially, the log will support OIDC tokens signed by classical crypto or PQC. Once there is widespread adoption of ML-DSA signed tokens, we will disallow any classically signed tokens. This is discussed more [below](#oidc-identity-provider).

The API will return a [c2sp.org/tlog-proof](http://c2sp.org/tlog-proof) (more discussed [below](#bundle-with-c2sp.org/tlog-proof)).

* Rekor will not include the canonicalized body. Clients must recompute the leaf hash from provided data, root of trust, credential, and claim key/value pairs.  
* The proof will include the token claims as opaque `extra` data. Token claims are included as an unordered list of elements matching `key:value` separated by `\n` (`0x0a`). Both `key` and `value` are strings. The `key` is the name of the context category as specified in the leaf specification. When the credential is an identity token, all `value` strings will be the parsed identity and provenance values from the token. When the credential is a public key, the optional `value` will be the provided context as a hex-encoded digest.  
* When verifying, clients must require at minimum the identity and issuer claim values. The list of user-provided claims will be merged with the list of log-provided claims.

```proto
message CreateEntryRequest {
    // Must be 32 bytes and a digest of the original data
    bytes msg = 1;
    oneof credential {
        IDToken idtoken = 1;
        PublicKey pubkey = 2;
    }
}

message IDToken {
    // OpenID Connect identity token, as an encoded JWT comprised of
    // dot-separated header, payload and signature
    // Issuer must be in a log allowlist
    string id_token = 1;
}

message PublicKey {
    // RFC9881 PKIX ML-DSA public key
    // Accepts ML-DSA-44, ML-DSA-65, ML-DSA-87
    bytes pubkey = 1;
    // ML-DSA signature, computed over
    // c2sp.org/identity-transparency/v1 || 0x00 || SHA256(msg)
    bytes sig = 2;
    // Optional context string. Must be a 32-byte SHA-256 digest.
    optional bytes context = 3;
}

message CreateEntryResponse {
    // c2sp.org/tlog-proof inclusion proof and witnessed checkpoint
    // Extra data will be a newline (0x0a) separated unordered list
    // of context strings in the format key:value
    string tlog_proof = 1;
}
```

#### DSSE

There are presently two issues with signing DSSEs:

1. DSSEs expect signatures. We've proposed a scheme that produces a different format for a signature such that none of the DSSE libraries will support verification.  
2. For when artifact keys are used: DSSE specifies its own PAE.

We've chosen to design the bundle format to break down the DSSE into its payload type, payload and an optional signature, and the Sigstore SDKs will handle verification of the proposed signature formats.

When generating a DSSE and submitting it to the log, the client will use the Pre-Auth-Encoding (PAE) from the DSSE specification as the data, submitting a hash of the PAE as the message in the log upload request.

* With an OIDC token, no artifact signature is generated, as the log proof acts as a signature  
* With a signing key, the signature should be over `c2sp.org/identity-transparency/v1 || 0x00 || SHA256(SHA256(PAE))`, with the prefix signifying an intent to log.

When signing a DSSE without submitting to the log, the signature should be over `sigstore.dev/v1 || 0x00 || SHA256(SHA256(PAE))`, with the prefix signifying a lack of intent to submit to the log.

```proto
message DSSE{
        string payload_type = 1;
        bytes payload = 2;
        // The PAE used before signing is:
        // "DSSEv1" + SP + LEN(type) + SP + type + SP + LEN(body) + SP + body
        // When signing, data is the PAE.
        // Unset when submitting an identity token to the log.
        optional Signature = 3;
}

// Note that the DSSE Pre Auth Encoding (PAE) is used, but the signature
// scheme is Sigstore specific. The raw message is never signed, it's
// always hashed as there is a risk otherwise to transmit large payloads
// to an HSM.
message Signature {
        // The signature is computed over a domain separated message:
        // sigstore.dev/v1 || 0x00 || SHA256(SHA256(data)) if message is not
        // submitted to a log
        // c2sp.org/identity-transparency/v1 || 0x00 || SHA256(SHA256(data)) if message is submitted
        // to a log
        bytes signature = 1;
}
```

We also would like to work with the DSSE community to consider a special extension for tlog-proof signatures.

#### Certificates as Credentials {#certificates-as-credentials}

The log will not support certificates as a credential, both in public and private deployments. There are a few reasons why:

* Code signing certificates from Fulcio bind ephemeral keys to identities from identity tokens. If there is an identity token, there is no need to submit a certificate as a credential. Instead, the identity token can be submitted to the log.  
* Code signing certificates from private PKIs that do not conform to Fulcio's certificate profile will be issued for managed keys. Since the keys are stable, monitorable identifiers, the key and artifact signature can simply be submitted to the log.  
* Code signing certificates can be issued from Fulcio binding *managed keys* to identities. However as noted above, the key would be a stable identifier, and so the key can be submitted to the log.  
* For private deployments that host an instance of Fulcio and Rekor, with this new proposal, Rekor is a superset of Fulcio and so deployers can simply host the log.  
* For private deployments that only host an instance of Fulcio, this discussion is moot since there is no log.

If there is a use case that we have missed, please reach out to discuss more. We want to avoid unnecessary complexity until there is a clear requirement.

We include some notes on how to support this below:

* The C2SP specification would need to be updated to include certificates as a `root of trust`. The `root of trust` will be a CA certificate root subject, as a raw string (not ASN.1 encoded).  
* For the API:  
  * We would require the full chain be provided since the log leaf format includes the root CA certificate as the root of trust. The log must limit the chain to a reasonable number of certificates to prevent a DoS vector.  
  * Not accepting the full chain is arguably an issue for monitoring with the current design. A malicious actor can create noise for a monitor by generating self-signed certificates containing any identity. Since the leaf contains the full certificate, a monitor for the public log *should* be configured with all trusted chains, defaulting to the public instance's trust root.  
  * An open question is what set of extensions should be recorded in the leaf. Like with processing JWTs, we want to be prescriptive on the set of known OIDs, both for building monitors and to prevent a DoS vector. At minimum, we should include provenance values. We could also include well-known code signing extensions like from Authenticode. This would be codified in the C2SP specification.

### Supported Client Signing Paths

The client will support three signing paths:

* Transparent signatures. The client submits a credential, either an identity token or managed signing key, to a transparency log, and the proof acts as an artifact signature.  
  * A key may also be bound to a certificate that does not conform to Fulcio's certificate profile. In this case, a client should submit the signature and key, not the certificate, to the log. The key is a stable, monitorable identifier.  
* PKI signatures, exclusively for private deployments (discussed more [below](#private-deployments)). To sign an artifact, the client requests a Fulcio certificate binding an ephemeral signing key and identity token, and timestamps the signature.  
* Managed signing key signatures. The key is either stored locally, in a KMS, or on an HSM or hardware token. A client can sign an artifact without any additional infrastructure. For private deployments, a client can optionally timestamp the signature to enable key rotation.

All three signing paths will be covered under the Sigstore conformance test suite. Not all SDKs may support all three signing paths. All SDKs must support transparent signatures with an identity token credential.

### Bundle with [c2sp.org/tlog-proof](http://c2sp.org/tlog-proof) {#bundle-with-c2sp.org/tlog-proof}

All Sigstore clients support the [protobuf bundle format](https://github.com/sigstore/protobuf-specs/blob/main/protos/sigstore_bundle.proto) for the signature and verification materials. Given that we will move to a log proof as a signature, we should also adopt [c2sp.org/tlog-proof](http://c2sp.org/tlog-proof) which standardizes the format of a log index, inclusion proof, and witnessed checkpoint. Token claims would be distributed in the opaque `extra` data. Token claims are included as an unordered list of string elements matching `key:value` separated by `\n` (`0x0a`).

We have previously discussed making a breaking change to the bundle format, e.g. cleaning up duplicated and unused values in `dev.sigstore.rekor.v1`. Leveraging [c2sp.org/tlog-proof](http://c2sp.org/tlog-proof) allows us to build and reuse tools across transparency ecosystems. We will replace the `TransparencyLogEntry` message with tlog-proof.

We could use only tlog-proof as an alternative to the Sigstore bundle. However, we distribute DSSEs in bundles and we want to support certificates that would not use proofs at all. Therefore we'll keep the bundle format as a wrapper, even if the bundle contains only a tlog-proof string.

#### Bundle Format

```proto
syntax = "proto3";
package dev.sigstore.bundle.v2;

import "google/api/field_behavior.proto";

// https://raw.githubusercontent.com/secure-systems-lab/dsse/9c813476bd36de70a5738c72e784f123ecea16af/envelope.proto
import "envelope.proto";
import "sigstore_common.proto";
import "sigstore_rekor.proto";

option go_package = "github.com/sigstore/protobuf-specs/gen/pb-go/bundle/v2";
option java_package = "dev.sigstore.proto.bundle.v2";
option java_multiple_files = true;
option java_outer_classname = "BundleProtoV2";
option ruby_package = "Sigstore::Bundle::V2";

message TimestampVerificationData {
        repeated dev.sigstore.common.v1.RFC3161SignedTimestamp rfc3161_timestamps = 1;
}

// Note that the DSSE Pre Auth Encoding (PAE) is used, but the signature
// scheme is Sigstore specific. The raw message is never signed, it's
// always hashed as there is a risk otherwise to transmit large payloads
// to an HSM.
message Signature {
        // The signature is computed over a domain separated message:
        // sigstore.dev/v1 || 0x00 || SHA256(SHA256(data)) if message is not
        // submitted to a log
        // c2sp.org/identity-transparency/v1 || 0x00 || SHA256(SHA256(data)) if message is submitted
        // to a log
        bytes signature = 1;
}

message Embedded {
        string payload_type = 1;
        bytes payload = 2;
        // The PAE used before signing is:
        // "DSSEv1" + SP + LEN(type) + SP + type + SP + LEN(body) + SP + body
        // When signing, data is the PAE.
	 // Unset when submitting an identity token to the log.
        optional Signature = 3;
}

message MessageSignature {
        dev.sigstore.common.v1.HashOutput message_digest = 1 [(google.api.field_behavior) = REQUIRED];
        // When signing, data is the artifact.
	 // Unset when submitting an identity token to the log.
        optional Signature signature = 2;
}

message TlogProof {
        // The log proof, formatted per c2sp.org/tlog-proof.
        // If OIDC based submit is used, the `extra` field contains the
        // relevant claims needed for recreation of the leaf during
        // verification. They are stored as an unordered list of elements
        // matching key:value separated by `\n` (0x0a). Both key and value
        // are strings.
        // If a signature is part of the leaf, verifiers MUST verify the
        // the artifact's signature, and compare that the signature matches
        // what's submitted to the log.
        bytes proof = 1 [(google.api.field_behavior) = REQUIRED];
        // If the artifact was signed prior to log submission.
        // Signature is over c2sp.org/identity-transparency/v1 || 0x00 || SHA256(SHA256(data))
        optional dev.sigstore.common.v1.PublicKeyIdentifier public_key = 2;
}

message PublicKey {
        dev.sigstore.common.v1.PublicKeyIdentifier public_key = 1 [(google.api.field_behavior) = REQUIRED];
        // Enables key rotation when signature is not logged
        optional TimestampVerificationData timestamps = 2;
}

message Cert {
        oneof cert_data {
                dev.sigstore.common.v1.X509Certificate certificate = 1 [(google.api.field_behavior) = REQUIRED];
                dev.sigstore.common.v1.X509CertificateChain certificate_chain = 2 [(google.api.field_behavior) = REQUIRED];
        }
        TimestampVerificationData timestamps = 3 [(google.api.field_behavior) = REQUIRED];
}

message VerificationMaterial {
        oneof content {
                // Signed with a raw key pair
                PublicKey public_key = 1 [(google.api.field_behavior) = REQUIRED];
                // Signed with cert + rfc3161 timestamp
                Cert cert = 2 [(google.api.field_behavior) = REQUIRED];
                // Logged to Rekor V3, authenticating to the log with an
		   // identity token or public key/signature
                TlogProof tlog_proof = 3 [(google.api.field_behavior) = REQUIRED];
        }
}

message Bundle {
        // MUST be application/vnd.dev.sigstore.bundle.v2.0+json when
        // when encoded as JSON.
        string media_type = 1;

        VerificationMaterial verification_material = 2 [(google.api.field_behavior) = REQUIRED];
        oneof content {
                MessageSignature message_signature = 3 [(google.api.field_behavior) = REQUIRED];
                Embedded embedded_data = 4 [(google.api.field_behavior) = REQUIRED];
        }
}
```

### sigstore-go & other SDKs

A new top-level API will be implemented that supports this new signing and verification flow. Signature verification should be done through sigstore/sigstore and theupdateframework/go-tuf which will use [filippo.io/mldsa](http://filippo.io/mldsa). Other ecosystems will be responsible for finding a suitable, well-tested and production-ready ML-DSA implementation.

There will be 4 supported flows.

* Transparent signatures with OIDC tokens, i.e. "keyless" signing  
* Transparent signatures with managed keys  
* PKI signatures with OIDC tokens  
* Managed signing key signatures

Note that for each of these flows, SDKs will support signing both artifacts and DSSE attestations. For artifacts, the message to sign is `SHA256(artifact)`. For DSSEs, the message is `SHA256(PAE)`.

#### Transparent signatures with OIDC tokens, "keyless" signatures

Signing flow for default "keyless" signing:

1. Client computes `m=SHA256(artifact)`  
2. Client initiates an OpenID Connect authentication flow and receives an identity token `tok`, or provides a token fetched out of band.  
3. Client sends `m` and `tok` to Rekor.  
4. Rekor parses `tok` and extracts the issuer `iss`. Rekor fetches `iss` public keys from a well-known endpoint and caches them locally. If `iss` is not in an allowlist, Rekor rejects the token.  
5. Rekor verifies `tok` using the public keys. Rekor rejects unverifiable tokens.  
6. Rekor extracts all relevant claims from `tok`, in particular `sub`/`email` and CI-specific claims. Rekor will standardize claims across CI providers using the [same mapping](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md#mapping-oidc-token-claims-to-fulcio-oids) from Fulcio.  
7. Rekor generates a leaf hash as per the [identity-based transparency log spec](https://github.com/C2SP/C2SP/pull/244). Rekor provides this hash to the Tessera backend.  
8. Once integrated and a witnessed checkpoint is generated, Rekor requests an inclusion proof. Rekor constructs a [c2sp.org/tlog-proof](http://c2sp.org/tlog-proof) with the proof, checkpoint, and token claims, and returns it to the client.  
9. Client returns proof to caller.

Verification flow:

1. Client loads trust root from PQC TUF root  
2. Client verifies checkpoint:  
   1. Signatures from log and witnesses  
   2. Meets defined [witness policy](https://github.com/C2SP/C2SP/pull/233)  
3. Client mandates user provides `subject` and `issuer`. Optionally, users may provide additional provenance values.  
4. Client computes leaf hash as per the [spec](https://github.com/C2SP/C2SP/pull/244), with `0x00{32}` as the receipt digest. User-provided claim values must override values parsed from the `extra` data.  
   1. `0x00{32}` denotes a receipt-less inclusion proof. For OIDC token signatures, no receipt will be provided. We discuss later on future work to provide a zero-knowledge proof as a receipt for processing a valid OIDC token.  
5. Client verifies inclusion proof

##### Note on Token Replay

To prevent token replay, clients could bind the token to a specific artifact, for example by setting the token nonce to a value such as `"sigstore" || m`. However, this is not feasible for two reasons:

* Many OpenID providers (e.g. GitHub Actions) do not support customizing the nonce.  
* A client may also want to reuse the token for signing multiple artifacts. For replay-resistant batch signing, a client could generate a Merkle tree of artifacts, set the nonce to the root of the tree, and then provide an inclusion proof for that artifact. This of course would add complexity. If we did not support batch signing, replay resistance would have to be optional.

#### Transparent signatures with managed keys

Signing flow for managed keys:

1. Client is configured to use `priv`/`pub`  
2. Client computes `m=SHA256(artifact)`  
3. Optionally, using a user-provided context value, client computes `ctx = SHA256(context value)`  
4. Client generates ML-DSA signature `s` with `priv` over `c2sp.org/identity-transparency/v1 || 0x00 || SHA256(m)`. If `ctx` is set, the signature is over `c2sp.org/identity-transparency/v1 || 0x00 || SHA256(m) || SHA256(SHA256("context")) || SHA256(ctx)`  
5. Client sends `s`, `pub` and `m` and optionally `ctx` to Rekor  
6. Rekor computes `d = c2sp.org/identity-transparency/v1 || 0x00 || SHA256(m)`, or with `ctx`, `d = c2sp.org/identity-transparency/v1 || 0x00 || SHA256(m) || SHA256(SHA256("context")) || SHA256(ctx)`, and runs `ML-DSA-Verify(d, s, pub)`. Rekor rejects any entry with an invalid signature.  
7. Rekor generates a leaf hash as per the [identity-based transparency log spec](https://github.com/C2SP/C2SP/pull/244), with the root of trust being `SHA256(pub)`. Rekor provides this hash to the Tessera backend.  
8. Once integrated and a witnessed checkpoint is generated, Rekor requests an inclusion proof. Rekor constructs a [c2sp.org/tlog-proof](http://c2sp.org/tlog-proof) with the proof, checkpoint, and token claims, and returns it to the client.  
9. Client returns proof to caller

Verification flow:

1. Client loads trust root from PQC TUF root, and is provided `pub` out of band  
2. Client verifies checkpoint:  
   1. Signatures from log and witnesses  
   2. Meets defined [witness policy](https://github.com/C2SP/C2SP/pull/233)  
3. Client verifies the "receipt", the artifact signature `s` with `pub` over `c2sp.org/identity-transparency/v1 || 0x00 || SHA256(SHA256(artifact))`, or when client is configured with a context string, `c2sp.org/identity-transparency/v1 || 0x00 || SHA256(m) || SHA256(SHA256("context")) || SHA256(SHA256(context string))`  
4. Client computes leaf hash as per the [identity-based transparency log spec](https://github.com/C2SP/C2SP/pull/244)  
5. Client verifies inclusion proof

#### PKI signatures with OIDC tokens

Note that this flow mandates signing with ephemeral keys only using Fulcio. If a user wants to sign with a managed key that is also bound to a certificate, use the managed signing key flow documented below.

Signing flow:

1. Client computes `m=SHA256(artifact)`  
2. Client initiates an OpenID Connect authentication flow and receives an identity token `tok`, or provides a token fetched out of band.  
3. Client generates ephemeral keypair `ePriv`/`ePub`  
4. Client generates a proof of possession `pop`, signing the `sub` of `tok` with `ePriv`  
5. Client sends `tok`, `pop`, and `ePub` to Fulcio  
6. Fulcio verifies `pop` using the `sub` from `tok` and `ePub`  
7. Fulcio generates certificate, binding `ePub` to claims from `tok`, including subject `sub`, issuer `iss` and optional provenance claims  
8. Fulcio returns ML-DSA-signed certificate `cert`  
9. Client generates ML-DSA signature `s` with `ePriv` over `sigstore.dev/v1 || 0x00 || SHA256(m)`. Note the different prefix to denote a lack of logging.  
10. Client deletes `ePriv`  
11. Client timestamps the signature by submitting `s` to an RFC3161-compliant Timestamping Authority (TSA)  
12. TSA issues ML-DSA-signed signed timestamp `ts` over `s`

Verification flow:

1. Client load trust root with Fulcio and TSA certificate chains  
2. Client verifies `ts` with TSA certificate chain, extracting timestamp  
3. Client verifies `cert` using Fulcio certificate chain and timestamp for validity  
4. Client verifies `cert` subject and issuer match user-provided subject and issuer. Users may also provide additional provenance claims to verify.  
5. Client extracts `ePub` from `cert`  
6. Client verifies `s` with `ePub` over `sigstore.dev/v1 || 0x00 || SHA256(SHA256(artifact))`

#### Managed signing key signatures

Signing flow:

1. Client is configured to use `priv`/`pub`  
2. Client computes `m=SHA256(artifact)`  
3. Client generates ML-DSA signature `s` with `priv` over `sigstore.dev/v1 || 0x00 || SHA256(m)`. Note the different prefix to denote a lack of logging.  
4. Optionally, client timestamps the signature by submitting `s` to an RFC3161-compliant Timestamping Authority (TSA)  
5. TSA issues ML-DSA-signed signed timestamp `ts` over `s`

Verification flow:

1. Client is provided `pub` out of band  
2. Client verifies artifact signature `s` with `pub` over `sigstore.dev/v1 || 0x00 || SHA256(SHA256(artifact))`  
3. Optionally, if the signature is timestamped:  
   1. Client loads trust root with TSA certificate chain  
   2. Client verifies signed timestamp `ts` using TSA certificate chain  
   3. Client verifies timestamp within `pub` validity period

### OIDC Identity Provider {#oidc-identity-provider}

We propose accepting either classically signed or PQC identity tokens. Unfortunately, there has been almost no movement around updating OpenID Connect for PQC. The specification is flexible and allows any algorithm identifiers, but implementers are unlikely to support PQC any time soon without the specification explicitly defining an identifier.

We have chosen to not block on this, as there are no known timelines for OpenID Connect adopting ML-DSA. Once a significant number of identity providers are issuing ML-DSA signed tokens, we can announce the deprecation of any non-PQC identity providers for the public instance and begin to accept only ML-DSA signed tokens.

The threat model does not change if Rekor accepts classical or PQC tokens. If the token were maliciously generated either due to a compromised token issuer or a quantum computer deriving the private key from a signature, in either case the usage of that token would still be auditable. The difference between these two scenarios is that the attack surface expands from a compromise of an issuer to anyone with a CRQC.

We will engage with public identity providers where we have an existing community connection, starting with Google for user and workload identity credentials and GitHub for CI credentials. We will primarily focus on CI identity providers. CI identity providers do not have to adhere to the OpenID Connect specification, as they only need to issue OIDC-compliant tokens and provide an OIDC-compliant public keys endpoint.

In order to support identity federation with Dex, we could create a temporary fork of Dex and patch in support for ML-DSA. We will need to also fork go-jose to implement the [ML-DSA JOSE spec](https://datatracker.ietf.org/doc/rfc9964/), though support for ML-DSA is [planned](https://github.com/go-jose/go-jose/issues/194). However, supporting ML-DSA in Dex does not improve the overall threat model, as Dex would still be accepting classically signed tokens. We will deprioritize this work and not include it in the scope of an MVP.

### TUF

We will create a second TUF repository to house a new TUF root signed exclusively with ML-DSA. We will not use the existing production root to distribute ML-DSA infrastructure keys, nor will we use the new TUF root to distribute classical crypto keys. These TUF repositories will be independent of one another. This means that we will have to hold a root signing event, streamed live, recorded, and if possible completed in person.

[Work is already underway](https://github.com/theupdateframework/taps/pull/195/) to update the TUF specification for ML-DSA. The linked PR proposes TAP21, with a scheme and keytypes for ML-DSA, along with a pre-hashing scheme to hash data before signing. TUF clients (e.g. go-tuf, python-tuf) will need to be updated to support TAP21.

### Infrastructure

We will leverage all of the existing infrastructure work that was completed for Rekor v2 to deploy a multi-region PQC Rekor. There should be no changes needed to terraform-modules. We will need to decide if we publish a new Helm chart or if we create a major revision of the existing rekor-tiles chart. Publishing a new chart would allow us to continue to update the previous chart for private deployers.

The transport should also be quantum-resistant, so we will enable PQC key exchange algorithms for TLS in the [GCP load balancer](https://docs.cloud.google.com/load-balancing/docs/post-quantum-tls).

## External blockers

### Hardware tokens

There is a lack of support for PQC with YubiKey or other hardware tokens. Given that these keys create the root of trust for the public Sigstore deployment, we will need to identify a solution before we can publish a public PQC root of trust. Given that [YubiKey](https://www.yubico.com/blog/the-future-of-authentication-in-2026-insights-from-yubicos-experts/) has not publicly announced any timeline, we will pursue alternatives.

We are recommending using the [Tillitis TKey](https://www.tillitis.se/) as a hardware root of trust. TKey is an FPGA-based security token backed by Mullvad VPN. Tillitis has published an ML-DSA signing [application](https://github.com/tillitis/tkey-pq-device-signer) and [CLI](https://github.com/tillitis/tkey-pq-sign-cli) for the TKey. The signing process is slow, but for our use, speed is not at all a concern since signing is an infrequent offline process.

An alternative explored included leveraging Apple's Secure Enclave which supports ML-DSA-65 and ML-DSA-87. We put together a [demo](https://github.com/Hayden-IO/pqc-secure-enclave) to demonstrate signing with an enclave-backed key. Issues include that token touches are cached, and by default keys are overwritable by any process. We would need to obtain a macOS developer license to use specific app grants that let the application create an app-bound key.

Another alternative is to use HSM-backed or software-based ML-DSA keys from a cloud provider. GCP and AWS support all three ML-DSA parameter choices. However, due to the lack of cloud provider support, we could not have the necessary diversity to avoid a quorum of keys on a single cloud provider.

Yet another alternative is to continue using the YubiKey PIV modules until real PQC hardware appears, but as private key storage:

* An ML-DSA-44 private key should fit into a single PIV data slot  
* When offline, the key would be protected to the same standard as current keys  
* When in use, the private key would be loaded into memory, which is less secure than the current YubiKey usage

### OpenID Connect

As noted [above](#oidc-identity-provider), we will engage with public identity providers where we have an existing community connection, starting with Google for user and workload identity credentials and GitHub for CI credentials. Until then, we will support both classically signed and PQC signed identity tokens.

### Witnessing

[c2sp.org/tlog-witness](http://c2sp.org/tlog-witness) has been updated to support ML-DSA. As public ML-DSA witnesses come online as part of [witness-network.org](http://witness-network.org), we will integrate Rekor with these witnesses.

## Non-repudiation & Credential Verifiability {#non-repudiation-&-credential-verifiability}

Non-repudiation in the legal sense means that a party cannot deny their actions. With cryptography, this means that a signer cannot deny generating a digital signature. Asymmetric cryptography gives us this property \- the signer is solely in possession of a private key and a signature is proof that the key was used.

The current "keyless" design does not provide non-repudiation, since Fulcio can forge a certificate without a valid identity token. While this malicious action is auditable, it does not guarantee non-refutability. Consider the typical workflow where an ephemeral key is associated with an identity. Let's say the signer denies signing an artifact.

* The log claims that a valid signature was seen and presents that signature. However, that signature was over an ephemeral key which the signer is no longer in possession of.  
* The CA claims that it saw a valid token. However, it does not persist the token because the token is a secret credential.

If the CA could present the token, then Sigstore would not be held liable. Either the signer is lying and did actually generate that signature, or the identity provider is to blame.

The proposed PQC design removes the CA component, but still does not provide non-repudiation. The log cannot prove it received a valid token.

Even if we could somehow present the token, signed identity tokens can't be verified without a historical OIDC provider key log. OIDC providers frequently rotate their keys (24 hours \- a few weeks typically). We could populate such a log, though ideally it would be operated by a neutral third party or by the providers themselves. If it were operated by Sigstore, Sigstore's view of the keys would be trusted, though we could provide a single global view with a witnessed log.

One idea would be to store the token signature off-tree in content-addressable-storage mapping Sa leaf hash to a token signature. If the signer were to deny having generated a message signature, the log would retrieve the token signature and the set of keys from the provider key log to prove the log saw a valid token. This is of course problematic in that the signature is meant to be a secret, and the log is now on the hook for securely managing a sensitive cache. Tokens do expire quickly so it's not as risky as storing long-lived credentials, but it's still quite risky.

An alternative is to present a zero-knowledge proof over a signature. This is a solved problem for RSA signatures by generating a [Guillou-Quisquater (GQ) signature](https://dl.acm.org/doi/abs/10.5555/88314.88372). However, research has just begun for ZKPs for ML-DSA signatures \- [ML-DSA Zero-Knowledge Proof Literature](https://docs.google.com/document/d/1ckp6yZl3lCSvvoWgBwY33huFcWumKlNNxlHZX9UTeOE/edit?resourcekey=0-fuzcpQSg6QY4tjFddjOTvA&tab=t.0). Some possible approaches include [Lantern](https://eprint.iacr.org/2024/457.pdf), [SmallWood](https://csrc.nist.gov/csrc/media/Projects/threshold-cryptography/documents/TCall-1/SmallWood-PW01.pdf), and the [circuit from the Longfellow library](https://github.com/google/longfellow-zk/tree/main/lib/circuits/tests/pq/ml_dsa), however given our lack of knowledge in this space, we will need to collaborate closely with researchers.

Removing the CA and having the log directly verifying the token does not change the threat model, it simply makes this lack of non-repudiation more evident. While we will not tackle this work as part of PQC Sigstore, we would like to experiment with it later on.

## Pseudonymous Signing

Signers who generate content from private repositories do not want to commit signatures to transparency logs which leak repository identifiers. Currently, these signers can either deploy their own private infrastructure, or use the public infrastructure but without Rekor. However, with the current infrastructure when Rekor was not logged to, maybe unbeknownst to the signer, private repository details were included in the code signing certificate written to the certificate transparency log.

With this new design, the public transparency log should now be acceptable to use. Now, without Fulcio and with the [identity-based transparency log spec](https://github.com/C2SP/C2SP/pull/244) leaf entry, private details will be masked. All identities and provenance data are hashed. Note that this is pseudonymization and is not truly privacy-preserving:

* Linkability between two signing events with the same identity hash is possible, which also leaks statistical metrics like the frequency of signing by identity hash.  
* Values that are guessable, such as monotonic integers, can still be derived.

## Private Deployments {#private-deployments}

In addition to the public instance, Sigstore infrastructure is deployed within private organizations, some of which have unique PKI needs. Private deployments typically either don't need or want transparency, as there are no external verifiers nor would any entity audit the log. Additionally, signers who generate content from private repositories do not want to commit signatures to transparency logs which leak identifiers.

Currently, the recommendation for private deployments is to deploy an instance of Fulcio and a timestamp authority alongside a private identity provider. An ephemeral signing key is generated and certified by the CA, an artifact signature is generated, and the signature is timestamped. Sigstore clients support this configuration, where transparency log proofs are optional.

We need to provide a solution for private deployments for PQC. We propose one of two options:

1. Use the same setup as the public deployment, with only a transparency log.  
   1. The log supports token identities for "keyless" signing, and public keys for auditing signature generation with managed keys. As discussed [above](#certificates-as-credentials), a private log deployment would not need certificate support.  
   2. The log deployment should be hardened exactly as a deployer would harden a CA instance, with minimal engineer access and a sufficient trust boundary separation.  
   3. The log would not be witnessed, which means split-views are possible. However, this is no different from a deployment with a CA without transparency. Additionally, this opens up the door to a log being witnessed. A private log can be witnessed without providing the log leaf contents, so there is no leakage of private information, even if it's just hashes in the leaf. There are active discussions in the [witnessing community](https://witness-network.org) about paid offerings to witness private logs, and organizations that want their private logs witnessed should reach out to the community.  
2. Support a code signing certificate setup with a CA and timestamp authority with no log.  
   1. Concretely, this means that a signing event generates 1 artifact signature, 1 ephemeral public key, 1 CA signature, and 1 signed timestamp. For ML-DSA signatures, this would produce a large signature bundle.  
   2. Go will support [RFC9881](https://datatracker.ietf.org/doc/rfc9881/) algorithm identifiers in Go 1.27. Temporarily, we would have to fork parts of `crypto/x509` to support ML-DSA certificate generation and parsing.  
   3. Additionally, we would have to update the core timestamping libraries for [RFC9882](https://datatracker.ietf.org/doc/rfc9882/).  
   4. Web PKI is making significant changes by adopting a radically different approach to PKIs for TLS using [Merkle Tree Certificates](https://datatracker.ietf.org/doc/draft-ietf-plants-merkle-tree-certs/) (MTC). MTC is an option for private deployments (discussed in [this talk](https://www.youtube.com/watch?v=0nQQbj5SSlA&t=4817s)). However MTC comes with a lot of unnecessary complexity for private code signing, such as landmarks and subtrees. MTC was designed specifically for Web PKI needs, and so we are reluctant to adopt it in Sigstore SDKs.

Note that (1) requires more infrastructure (a database), whereas (2) requires two services but each is trivially scalable. We will seek feedback from private deployers if log infrastructure is acceptable.

The suggestion for private deployments at this time is to **deploy a private transparency log**. We will work with known private deployers to better understand their requirements and identify any unique needs where this setup wouldn't work.

We also recognize the value of simple-to-deploy services, and while we do think Tessera-based logs are easy to deploy and maintain, we are aware of some organizations that do not want to operate logs yet or that already have operational experience with maintaining CAs.  Concurrently, we will update Fulcio and the Timestamp Authority to support ML-DSA certificates and timestamps. Note that there are no plans to have a public deployment of Fulcio and the TSA, these services would be exclusively for private deployments.

Summarizing, there would be three signing paths supported by clients:

* Transparent signatures, for both public and private deployments, where the client submits a credential (identity token or key) to a transparency log and the proof acts as an artifact signature  
* PKI signatures, for private deployments, where the client exchanges an identity token for a certificate and timestamps the signature  
* Managed keys, where a client signs an artifact. For private deployments, optionally timestamping the signature

## Migration for the Public Instance

The public instance will remain active until **2029**, unless there are further advancements in a CRQC at which point the timeline will be expedited. The public instance maintainers will commit to maintaining the infrastructure, including bug and security fixes, though we do not plan to develop any new features. We will announce the 2029 turndown of the public instance when we launch a production-ready PQC public instance.

The production-ready PQC public instance is currently targeted for mid-2027. Pending client development, we may be able to expedite this to late 2026 or early 2027\.

The public instance will not switch clients to automatically use the PQC instance. Using the PQC instance will be opt-in, where users configure their clients to use the PQC TUF root. After the production-ready PQC instance is deployed, around mid-late 2027 we will release new major client revisions that default to using the PQC instance. We will continue to support older client versions, but will encourage adoption of the new client version that supports signing and verification with ML-DSA.

### Existing Rekor v2 Deployment

Clients will continue to support Rekor v2. The public instance will continue to host the existing Rekor v2 log, but we will not update the signing configuration to switch clients over to writing to Rekor v2 by default. We will leave the Rekor v2 public key in the trusted root. We will work with high QPS signers to migrate over to Rekor v2 with a custom signing configuration if Rekor v1 is unable to scale. We could provide a second signing configuration file in the public instance which specifies the latest Rekor v2 URL. If the Rekor v2 instance sees significant usage, we will rotate the log periodically, at which point we will instruct high QPS signers to use this second signing configuration to automatically pick up new log shards.

## Alternatives Considered

### Maintain current specification, replacing ECC with ML-DSA

This solution would make the minimal amount of changes to support PQC, focused primarily on cryptographic agility with minimal client changes. The signing and verification workflows would remain identical, except any usage of ECC would be replaced with ML-DSA.

Replacing every existing ECC signature with an ML-DSA signature is technically feasible. However:

* This is still a breaking change for ecosystems, where signature verifiers must update in advance of signers. If we're going to make a breaking change, we should aim to mitigate all concerns concurrently and not introduce another breaking change later on.  
* We still have to contend with large leaf entries and bundles.

### Hybrid bundles

This solution would integrate PQC into existing bundles, with a client signing an artifact twice, once with ECC and once with ML-DSA. Cryptographers are [recommending against](https://words.filippo.io/crqc-timeline/#:~:text=Hybrid%20classic%20%2B%20post%2Dquantum%20authentication%20makes%20no%20sense) this proposal, though constrained environments like finance that are not as agile may still try to adopt the proposed [hybrid specification](https://datatracker.ietf.org/doc/draft-ietf-lamps-pq-composite-sigs/).

Supporting hybrid bundles means exposing verification configuration to accept one or both of ECC and ML-DSA. This adds complexity and an opportunity for fallback attacks. There is no motivation to go explore this option further for Sigstore.

### Dual Signed Checkpoints

Outlined in [https\://github.com/sigstore/rekor-tiles/issues/425](https://github.com/sigstore/rekor-tiles/issues/425)

This solution proposed signing a Rekor checkpoint with both ECC and ML-DSA. This proposal enables a slower transition to PQC, where we can trust artifacts signed with ECC from before when a CRQC was available.

However, this proposal was always meant as a temporary solution to trust non-PQC signatures. Once a CRQC is available, we would have to migrate to another proposal. Therefore, we will not pursue this solution as it would eventually be obsoleted.

### Merkle Tree Certificates

This solution would implement [Merkle Tree Certificates](https://datatracker.ietf.org/doc/draft-ietf-plants-merkle-tree-certs/), a solution from the WebPKI space to reduce the number of signatures needed to verify certificates while also supporting Certificate Transparency (CT). As mentioned above, MTC is a [possible solution](https://www.youtube.com/watch?v=0nQQbj5SSlA&t=4817s) for replacing PKIs. However, MTC comes with additional complexity designed to optimize Web PKIs (e.g. landmarks, subtree cosigning) that Sigstore would not need. Our tooling would have to be built around another community's specifications and tools, adding another potential external blocker for being able to progress.
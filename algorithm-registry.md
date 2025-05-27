# Algorithm Registry

This file is designed to act as a source of truth regarding what signing
algorithms are recommended across the Sigstore ecosystem. Any changes to this
file **must** be reflected in the `PublicKeyDetails` enumeration in
[`sigstore_common.proto`] in [sigstore/protobuf-specs].

Sigstore clients aren't required to support all algorithms in this registry,
and **MAY** support algorithms that aren't in the registry. However,
compatibility with the Sigstore Public Good Instance requires support
for at least one of these algorithms.

## Signature Algorithms

| Algorithm | Name                       | Usage       | Notes                                                                            |
|-----------|----------------------------|-------------| -------------------------------------------------------------------------------- |
| RSA       | rsa-sign-pkcs1-2048-sha256 | verify only | Not recommended.                                                                 |
|           | rsa-sign-pkcs1-3072-sha256 | sign/verify |                                                                                  |
|           | rsa-sign-pkcs1-4096-sha256 | sign/verify |                                                                                  |
|           | rsa-sign-pss-2048-sha256   | verify only | Not recommended.                                                                 |
|           | rsa-sign-pss-3072-sha256   | sign/verify |                                                                                  |
|           | rsa-sign-pss-4096-sha256   | sign/verify |                                                                                  |
| ECDSA     | ecdsa-sha2-256-nistp256    | sign/verify |                                                                                  |
|           | ecdsa-sha2-384-nistp384    | sign/verify |                                                                                  |
|           | ecdsa-sha2-512-nistp521    | sign/verify |                                                                                  |
| EdDSA     | ed25519                    | sign/verify |                                                                                  |
|           | ed25519-ph                 | sign/verify | Recommended only for `hashedrekord`.                                             |
| LMS       | lms-sha256                 | sign/verify | Stateful; signer selects the `H` parameter. Not recommended for keyless signing. |
| LM-OTS    | lmots-sha256               | sign/verify | One-time use only; signer selects `n` and `w`.                                   |
| ML-DSA    | ml-dsa-65                  | sign/verify | Experimental; Pure variant.  Not yet fully functional.                           |
|           | ml-dsa-87                  | sign/verify | Experimental; Pure variant.  Not yet fully functional.                           |

### Parameter configuration for LMS and LM-OTS

LMS and LM-OTS are both hash-based signature schemes. Both require the signing party
to make parameter choices during key generation.

In both cases, the selected parameters are encoded in the public key representation.
See [RFC 8554 S5.3](https://www.rfc-editor.org/rfc/rfc8554.html#section-5.3) for LMS and
[RFC 8554 S4.3](https://www.rfc-editor.org/rfc/rfc8554.html#section-4.3) for LM-OTS public key
formats. Additionally, see [RFC 8708 S4](https://www.rfc-editor.org/rfc/rfc8708.html) for
`SubjectPublicKeyInfo` and `AlgorithmIdentifier` encodings for both LMS and LM-OTS
public keys.

### ML-DSA and Post-Quantum Cryptography (PQC)

Since 2016, NIST has been accepting and refining nominations for PQC algorithms, culminating
in the release of [FIPS 204](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.204.pdf).  The
ML-DSA algorithms that are being integrated into Sigstore are their pure variants (as opposed
to HashML-DSA) and are currently preferred for quantum-resistant signing.  They are larger 
than classical signatures, making their deployment more costly.  Future PQC algorithms 
may be selected by NIST, and these will be considered as they are released.

⚠️  ML-DSA-65 and ML-DSA-87 are currently not fully operational within Sigstore.  This warning
will be removed when these algorithms are widely supported by Sigstore clients and servers, but
caution should be exercised in deployment.

## Hash Algorithms

Generally speaking, these hash algorithms are implied by the above signing suites.
However, clients *may* need to list or configure them explicitly, e.g. for custom
signing schemes or as part of a `hashedrekord` entry.

| Algorithm | Name         |
|-----------|--------------|
| SHA2      | sha2-256     |
|           | sha2-384     |
|           | sha2-512     |
| SHA3      | sha3-256     |
|           | sha3-384     |

[`sigstore_common.proto`]: https://github.com/sigstore/protobuf-specs/blob/main/protos/sigstore_common.proto

[sigstore/protobuf-specs]: https://github.com/sigstore/protobuf-specs

### History

See [Sigstore: Configurable Crypto Algorithms](https://docs.google.com/document/d/18vTKFvTQdRt3OGz6Qd1xf04o-hugRYSup-1EAOWn7MQ/)
specification for the design rationale for this registry.

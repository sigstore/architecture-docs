# Fulcio, A Certificate Authority for Code Signing 

This document describes Fulcio, a certificate authority for issuing short-lived code signing certificates for an OpenID Connect (OIDC) identity, such as an email address. 

## 1. Introduction

Artifact signing classically required the management of a signing key. Verification policies must specify a mapping between a verification key and the artifacts which the key can verify. Self-managed keys require:
   * **Distribution**: Distribution is the process of publicly describing the mapping between verification keys and artifacts. Typically as a verifier, this mapping is implicitly trusted on first use (TOFU), and then persisted for future verifications. 
   * **Storage**: Storage refers to how the signing key is managed. This might be on a personal flash drive, or in a cloud environment using a key management service. This requires protection of the key material, either through on-disk encryption or appropriate ACLs when stored in the cloud.
   * **Revocation**: If the signing key is lost or compromised, a revocation mechanism is required to mark the key as revoked for verifiers. Additionally, a new key must be distributed to clients. As a verifier, how do I trust an update to the mapping between key and artifacts, and differentiate between a valid update and an update due to a malicious compromise? Additionally, revocation is plagued with issues around indefinitely-growing revocation list size and difficulty in distributing these large and frequently updated lists to verifiers.

Sigstore aims to simplify signing by eliminating the need to manage long-lived signing secrets. Public key infrastructure (PKI) can be leveraged to reduce the need to self-manage artifact signing keys. A certificate authority can issue short-lived code signing certificates that bind an identity to a public key. 

For verifiers, the verification policy maps identities to artifacts rather than keys to artifacts. While keys are still used to sign and verify signatures, verifiers no longer need to maintain mappings between keys and artifacts. Code signing certificates are trusted through the PKI. If As long as the verifier trusts the root certificate in the PKI, then they can trust the code signing certificate and the artifact verification key embedded within it. Artifact signers only need to manager their identity, not the keys. Keys can become ephemeral, generated for a single signing event and then discarded.

### 1.1 Conventions and Definitions

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).


## 2. Overview

To issue code signing certificates, Sigstore uses Fulcio, a certificate authority that binds OpenID Connect identities to public keys and writes issued certificates to a publicly auditable transparency log. While Fulcio primarily focuses on OpenID Connect identities, it also offers support for other identity systems like SPIFFE and Kubernetes service accounts, for added flexibility. 

### 2.1. Timestamping

The certificates that Fulcio issues are short-lived. By using ephemeral keys and short-lived certificates, Fulcio avoids the need for revocation lists. To verify a short-lived certificate, a timestamping service is needed to verify that the issued certificate was valid during artifact signing. Sigstore clients can leverage either Sigstore's own timestamping service ([Spec: Timestamping Service](https://docs.google.com/document/d/1FoRHXejIhXwEai0RS3iRsN1HfCV16fJOp582Vl8KA7A/edit#heading=h.dal05eqvj5ql)) that implements [RFC 3161](https://www.ietf.org/rfc/rfc3161.txt) to provide a trusted timestamp.

A signed timestamp provides a record of a signing event. The signed timestamp MUST be created over the artifact signature to associate a timestamp with a signing event. This proves possession of an artifact signing key at time of signature issuance.

### 2.2. Identity

Fulcio is both a certificate authority and a registration authority, meaning that Fulcio handles authentication logic when issuing a certificate.

An identity in Sigstore refers to the pair of an identifier and an issuer. By itself, an identifier does not uniquely identify a signer. For example, consider email addresses. There is no guarantee that "user@gmail.com" as authenticated by Google is the same user as "user@microsoft.com" authenticated by Microsoft. The combination of identifier and issuer or provider uniquely identifies an entity or account. Identity providers attest to a given identity, which are commonly represented as usernames like "user1" or email addresses like "user1@domain.com". There is no guarantee that "user1" attested by one identity provider controls the same account attested by another identity provider, for example, if the username is common or if someone is squatting the username. Similarly, there is no guarantee that an identity represented by an email address is the same user across identity providers. For example, the identity provider may not check ownership of the email address when a user signs up with the provider. There are also no guarantees the account is still under control of the original user (and not in some manner compromised).

Overall, for Fulcio-issued certificates, a verifier MUST check both the identifier and issuer of the identifier.

Identities must be presented to Fulcio signed by a trusted party. The OpenID Connect protocol was chosen as the initial authentication mechanism for signers, considering that a user is likely to already have an identity on a well-known provider, and integration for other providers should be straightforward if they conform to the protocol.

#### 2.2.1. OpenID Connect

Identity is provided by the [OpenID Connect (OIDC) framework](https://openid.net/developers/how-connect-works/), an authentication extension to OAuth 2.0. Fulcio accepts OIDC ID tokens from a configured list of identity providers (IDPs). Before certificate issuance, Fulcio verifies the signed OIDC ID token, and extracts the identity from the ID token. Fulcio is designed to be easily extensible and can be configured with any IDP that conforms to the OIDC specification. Fulcio accepts a variety of identities, including email addresses, [SPIFFE IDs](https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/#spiffe-id), Kubernetes service accounts, CI (e.g. GitHub Actions, GitLab, Buildkite) workflow identities, and custom URIs and usernames. Identity is specified in the Subject Alternative Name ([SAN](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.6)). For CI, additional identity and provenance metadata is included in custom extensions as defined in [OID Info](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md).

In addition to the identity of the signer, Fulcio also includes the issuer of the identity in a certificate extension. To verify a certificate, a verifier MUST use both the identity and issuer. An identity represented by "example@domain.com" issued by identity provider google.com is different from an identity with the same email at github.com.

Fulcio also supports identities provided by federated issuers, such as [Dex](https://dexidp.io/), when operated in the same trust domain as the service. A federated identity provider offers OpenID Connect issuance for various identity specifications such as LDAP or OAuth2. It will issue a token signed by the federated provider with the same claims as the origin provider, along with a claim that specifies the origin provider. A federated identity provider also lets a service operator control the lifetime of the issued tokens, limiting the risk of a leaked token.

Fulcio-issued certificates do not distinguish between tokens provided directly by an issuer and by a federated issuer. Since the federated provider is operated in the same trust domain as the service, the federated provider is considered an implementation detail. If a client chooses to verify identity tokens before certificate issuance and inspect the claims of the token, then a client MUST be aware of the distinction between federated and unfederated tokens.

A credential SHOULD be a JSON Web Token (JWT) OpenID Connect ID token that conforms to the [OpenID Connect specification](https://openid.net/specs/openid-connect-core-1_0.html#IDToken). The JWT MUST contain:

   * `aud`, the audience of the token, set to \`sigstore\`
   * `iss`, the issuer of the token
   * `exp`, the expiration of the token
   * `iat`, the time of token issuance

The JWT must also include a claim to represent user identity. This is configurable for each supported provider. Common claims include `sub` and `email_address`.

#### 2.2.2. Machine Identity

Fulcio issues certificates not only for user-based identifiers but also machine identities. These can be specified through [SPIFFE IDs](https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/#spiffe-id), Kubernetes service accounts, or CI workflow identifiers. These identifiers will include information about the environment that the workflow occurred in. This could include the Kubernetes cluster, Cloud project, or source repository.

Using machine identities instead of human identifiers provides privacy benefits, as no user identity is associated with the signing event. It also can provide a stronger link between source and build if a machine identity from a CI workflow signs a build.

See [Spec: Fulcio](https://docs.google.com/document/d/1W5xp3g8_jaqzDQmIvepNYsWb-bQNc0U2ZQgQ700Kjok/edit#heading=h.lrl46qy6806e) for more information about required certificate extensions.

### 2.3. Keys

Note that Fulcio can be used with self-managed keys, as described in a [Sigstore blog post](https://blog.sigstore.dev/adopting-sigstore-incrementally-1b56a69b8c15). Fulcio does not enforce the use of ephemeral keys. The signer may have experience with managing keys and appropriately locking down the signing key, for example through the use of an HSM or ACLs. Fulcio still provides benefits through a tighter integration with Sigstore and simplifying the verification policy.

## 3. Issuance - Life of a Request

The client submits a certificate request to Fulcio. The certificate request MUST contain a certificate request described subsequently and OpenID Connect (OIDC) identity token. This is a signed JWT containing information about the principal (identity of the signer), the issuer (who issued the identity token - Google, Microsoft, GitHub, etc.) and additional metadata such as expiration. The principal identity can either be a signing identity in the form of an email or username, or a workload identity. The certificate request MUST contain either:

- A public key and signed challenge. This is the public portion of a cryptographic key pair generated by the client. The public key will be embedded in the issued X.509 certificate. The challenge proves the client is in possession of the private key that corresponds to the public key provided. The challenge SHOULD be created by signing the subject (`sub`) of the OIDC identity token.
- A PKCS#10 ([RFC2986](https://www.rfc-editor.org/rfc/rfc2986)) certificate signing request (CSR), which also provides a proof of possession and the public key. The CSR subject MAY contain the subject of the OIDC ID token, but there is no mandate to do so, as Fulcio will not check that the subject of the CSR matches the subject of the token.

Fulcio MUST authenticate the OIDC ID token. To authenticate, Fulcio MUST follow the [OIDC specification](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation), which includes the following steps:

- Use the issuer claim from the token to find the issuer's OIDC discovery endpoint
- Download the issuer's verification keys from the discovery endpoint
- Verify the ID token signature

Fulcio does not support MAC-based authentication.

Once the token has been authenticated, Fulcio MUST verify the client is in possession of the private key of the public key they’ve submitted. Fulcio MUST verify the signed challenge or CSR. For a signed challenge, this MUST be a signature over the identity claim of the ID token, which SHOULD be the `sub` claim but MAY be a non-standard claim as Fulcio supports configuration of this claim. The challenge and CSR are verified using the provided public key.

Fulcio now creates and signs a code signing certificate for the identity from the ID token. Fulcio MUST:

- Embed the provided public key in the certificate
- Set the certificate's Subject Alternative Name (who the certificate is issued for) to match the identity from the OIDC ID token. This could be an email, SPIFFE ID, or CI workflow identity
- Include the OIDC ID token issuer in a custom field in the certificate
- Set various X.509 extensions depending on the metadata in the OIDC ID token claims (e.g. CI workflow information)

Signing MAY include a call to a remote key management service, if the CA signing key does not live in-memory.

Fulcio SHOULD append the issued certificate to an immutable, append-only, cryptographically verifiable certificate transparency (CT) log, which allows for issuance to be publicly auditable. The certificate transparency log MUST return a Signed Certificate Timestamp ([SCT](https://datatracker.ietf.org/doc/html/rfc6962#section-3)). The SCT is a promise of inclusion in the log, signed by the CT log. It can be verified without accessing the log, though a client can also request a cryptographic proof of inclusion directly from the log. Fulcio MUST embed the SCT within the certificate when present.

Fulcio finally returns the certificate to the client.

## 4. Certificate Transparency

Fulcio maintains a certificate transparency (CT) log, writing all issued certificates to the log.
Users of Sigstore can verify via cryptographic proof that certificates are included in the log along with monitoring the log for inconsistencies.

Certificate transparency is critical to auditability of certificate issuance. Without certificate transparency, a certificate authority can mis-issue certificates, either due to a misconfiguration or maliciously, without detection. Verifiers of Fulcio-issued certificates MUST require a proof or promise that the certificate was appended to a log, except for private deployments. For private deployments of the CA, transparency may not be necessary if there already exists a mechanism for an audit log.

The CT log is backed by [Trillian](https://github.com/google/trillian), a highly scalable and verifiable data store. The certificate-transparency-go [library](https://github.com/google/certificate-transparency-go/tree/master/trillian) implements [RFC6962](https://datatracker.ietf.org/doc/html/rfc6962), the RFC for certificate transparency.

To verify an entry, a verifier must either query the log for an inclusion proof or trust a promise, which can be verified offline. [SCTs](https://datatracker.ietf.org/doc/html/rfc6962#section-3) represent a cryptographic promise that the CT server will include an entry in the log within a fixed amount of time. The SCT contains a signature over a timestamp and certificate. It is verified using the log's public key, which should be securely distributed out of band.

SCTs can either be embedded in a certificate or detached from the certificate. Fulcio prefers embedding SCTs in certificates. If an SCT is detached, this means that Fulcio returns the SCT alongside the certificate, and it's up to the caller to store the SCT.

### 4.1. Sharding

A CT log can grow indefinitely, only bounded by the size of the underlying database. A large CT log will take longer for auditors to verify, and will also increase the burden on log replicators. Therefore, log operators need to create log shards, where after a certain period, a new log is turned up and the old log is frozen, accepting no more entries.

A log operator MUST create new log shards periodically. If temporarily sharded, the log operator SHOULD include either the year or month in the log name, depending on the frequency. For more details refer to [Spec: Transparency Service](https://docs.google.com/document/d/1NQUBSL9R64_vPxUEgVKGb0p81_7BVZ7PQuI078WFn-g/edit#heading=h.6w69n885z90t) and [Spec: Sigstore Public Deployment](https://docs.google.com/document/d/1C0naS8dP-k5c8z-kKa06ijyunYc8hswIwTY1nPm2M5g/edit#heading=h.sa2gitou4fkm)

## 5. Threat Model

See [Sigstore Threat Model (for website)](https://docs.google.com/document/d/13-nbmKqMqTUhsHc9L6V0j_8AWb5O-7f1qSpNyVFHbdQ/edit#heading=h.gj8hr3dbhp5z) for a more thorough analysis.

### 5.1. Threats

We consider two threats, a malicious or misconfigured identity provider or CA.

A malicious identity provider can craft a cryptographically valid identity token that is accepted by Fulcio. This token will include identity claims when the referenced identity did not request a token.

A malicious CA can craft a cryptographically valid certificate that can be verified with a trusted root certificate. This certificate will include tampered identity claims. A malicious, or misconfigured, CA can choose to ignore values in the identity token when issuing a certificate. A malicious CA could also issue an extra certificate for each request, bound to a key it controls.

### 5.2. Mitigations

Certificate transparency mitigates the threats of malicious or misconfigured identity providers and CAs. Certificates are issued to publicly auditable, append-only logs. An identity owner MUST monitor the log for occurrences of their identity, and alert the ecosystem if they find any unexpected instances in the log. If the identity owner does not, then the CA could misissue certificates, or a malicious identity provider could issue a token that is used to issue a valid certificate.

A certificate verifier MUST check either an inclusion proof or promise of inclusion. The verifier will then know that the certificate is in a public log and that the identity owner can verify issuances. If the certificate is presented without a proof or promise, then the verifier should not trust it, since there is no guarantee that the certificate is publicly auditable.

In addition to the identity owner monitoring the log, there must exist a set of witnesses that verify the integrity and consistency of the log, verifying that the log remains append-only. This mitigates the risk of a malicious log operator that tampers with the log contents.

## 6. Architecture

Fulcio supports various signing backends that are responsible for key management, certificate generation and signing.
An identity service MUST sign certificates with either secured on-disk keys or a remote key management service.

### 6.1. Public Key Infrastructure

Fulcio supports a PKI where either Fulcio is an intermediate CA or a root CA. Fulcio SHOULD be an intermediate CA, so that the intermediate CA certificate can be rotated and shorter lived without changing a longer-lived root. In this model, the root CA SHOULD be offline and only accessed when requesting an intermediate certificate.

### 6.2. Signing

#### 6.2.1. KMS

The KMS signing backend uses cloud key management services (KMS) to generate a certificate signature. This requires setting up a KMS key with a cloud provider, such as AWS, GCP, Azure or Hashicorp Vault. On setup, the public key of the signer will need to be certified, providing a certificate chain. The CA can either run as an intermediate CA chaining up to an offline root CA, or as a root CA, though the KMS signing backend is primarily meant to be used as an intermediate CA.

#### 6.2.2. Tink

The [Tink](https://github.com/google/tink) signing backend uses an on-disk signer loaded from an encrypted Tink keyset and certificate chain, where the first certificate in the chain certifies the public key from the Tink keyset. The Tink keyset MUST be encrypted with a KMS key, and stored in a JSON format. Tink keysets use strong security defaults and are the most secure way to store an encryption key locally.

#### 6.2.3. Google Cloud Platform CA Service

The GCP CA Service signing backend delegates creation and signing of the certificates to a CA managed in GCP. You will need to create a DevOps-tier CA pool and one CA in the CA pool. This can either be an intermediate or root CA.

#### 6.2.4. PKCS#11 HSM

PKCS#11 Hardware Security Module (HSM) support in Sigstore Fulcio enhances the security and trustworthiness of digital signatures by leveraging specialized cryptographic hardware. By integrating PKCS#11, Fulcio ensures that private keys used for signing are stored and managed in a highly secure, tamper-resistant environment. PKCS#11 HSM offers robust safeguards against key compromise and unauthorized access.

#### 6.2.5. On-disk file - For testing

The on-disk file-based signing backend loads a password-protected private key and certificate chain, and also monitors for changes to either, reloading the key and chain without requiring a server reboot. This signer supports a CA as either a root or intermediate. This signer MUST NOT be used for production, since the private key is only password-protected and could be compromised with a weak password.

#### 6.2.6 Ephemeral CA - For testing

Ephemeral CAs create the key material in memory and destroy the key material on server turndown. Ephemeral CAs MUST NOT be used for production.

## 7. Certificate Profile

### 7.1. Root Certificate

A root certificate MUST:

   * Specify a Subject with a common name and organization
   * Specify an Issuer with the same values as the Subject
   * Specify critical Key Usages for Certificate Sign and CRL Sign
   * Specify critical Basic Constraints to `CA:TRUE`
   * Specify a unique, random, positive, 160 bit serial number according to [RFC5280 4.1.2.2](https://www.rfc-editor.org/rfc/rfc5280.html#section-4.1.2.2)
   * Specify a Subject Key Identifier
   * Be compliant with [RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)

A root certificate MUST NOT:

   * Specify other Key Usages besides Certificate Sign and CRL Sign
   * Specify any Extended Key Usages

A root certificate SHOULD:

   * Use the signing algorithm ECDSA NIST P-384 (secp384r1) or stronger, or RSA-4096
   * Have a lifetime that does not require frequent rotation, such as 10 years

A root certificate MAY:

   * Specify a Basic Constraints path length constraint to prevent additional CA certificates from being issued beneath the root
   * Specify an Authority Key Identifier. If specified, it MUST be the same as the Subject Key Identifier
   * Specify other values in the Subject

### 7.2 Intermediate Certificate

An intermediate certificate MUST:

   * Specify a Subject with a common name and organization
   * Specify an Issuer equal to the parent certificate's Subject
   * Specify critical Key Usages for Certificate Sign and CRL Sign
   * Specify an Extended Key Usage for Code Signing
   * Specify a lifetime that does not exceed the parent certificate
   * Specify critical Basic Constraints to `CA:TRUE`
   * Specify a unique, random, positive, 160 bit serial number according to [RFC5280 4.1.2.2](https://www.rfc-editor.org/rfc/rfc5280.html#section-4.1.2.2)
   * Specify a Subject Key Identifier
   * Specify an Authority Key Identifier equal to the parent certificate's Subject Key Identifier
   *  Be compliant with [RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)

An intermediate certificate MUST NOT:

   * Specify other Key Usages besides Certificate Sign and CRL Sign
   * Specify other Extended Key Usages besides Code Signing

An intermediate certificate SHOULD:

   * Specify a Basic Constraints path length constraint of 0, `pathlen:0`. This limits the intermediate CA to only issue end-entity certificates
   * Use the signing algorithm ECDSA NIST P-384 (secp384r1) or stronger, or RSA-4096
   *  Have a lifetime that does not require frequent rotation, such as 3 years

An intermediate certificate SHOULD NOT:

   * Use a different signature scheme (ECDSA vs RSA) than its parent certificate, as some clients do not support this
   * Specify the Extended Key Usage as critical

An intermediate certificate MAY:

   * Be optional. An end-entity certificate MAY be issued from a root certificate or an intermediate certificate. Clients MUST be able to verify a chain with any number of intermediate certificates.

### 7.3. Issued Certificate

An issued certificate MUST:

   * Specify exactly one GeneralName in the Subject Alternative Name extension, as a critical extension. It MUST be populated by either:
      * An email address as an rfc822Name GeneralName
      * A URI as a uniformResourceIdentifier GeneralName
      * A username as an otherName GeneralName with OID [`1.3.6.1.4.1.57264.1.7`](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md#1361415726417--othername-san)
   * Specify an Issuer equal to the parent certificate's Subject
   * Specify a critical Key Usage for Digital Signature
   * Specify an Extended Key Usage for Code Signing
   * Specify a lifetime that does not exceed the parent certificate
   * Specify a unique, random, positive, 160 bit serial number according to [RFC5280 4.1.2.2](https://www.rfc-editor.org/rfc/rfc5280.html#section-4.1.2.2)
   * Specify a Subject Key Identifier
   * Specify an Authority Key Identifier equal to the parent certificate's Subject Key Identifier
   * Specify an empty Subject
   * Be compliant with [RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)
   * Specify a public key that is either:
      * ECDSA NIST P-256, NIST P-384, or NIST P-521
      * RSA of key size 2048 to 4096 (inclusive) with size % 8 = 0, E = 65537, and containing no weak primes
      * ED25519
   * Specify the OpenID Connect identity token issuer in a domain-specific OID. The OID SHOULD be [`1.3.6.1.4.1.57264.1.8`](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md#1361415726418--issuer-v2). The OID MAY be [`1.3.6.1.4.1.57264.1.1`](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md#1361415726411--issuer), which is deprecated but clients SHOULD still support it for previously issued certificates. The difference between `1.3.6.1.4.1.57264.1.8` and `1.3.6.1.4.1.57264.1.1` is that the former is formatted to the [RFC5280](https://datatracker.ietf.org/doc/html/rfc5280#section-4.1) specification as a DER-encoded string, whereas the latter is a raw string.
   * Be appended to a Certificate Transparency log. Clients MUST NOT trust certificates that do not present either a proof of inclusion or a Signed Certificate Timestamp (SCT)

An issued certificate MUST NOT:

   * Specify a nonempty Subject
   * Specify multiple GeneralName values in the Subject Alternative Name extension
   * Specify other Key Usages besides Digital Signature
   * Specify other Extended Key Usages besides Code Signing


An issued certificate SHOULD:

   * Use an ephemeral key. A client MAY request a certificate with a long-lived key, but a client MUST adequately secure the key material
   * Append a precertificate to a Certificate Transparency log, where the precertificate MUST be signed by the certificate authority and MUST include a poison extension with OID `1.3.6.1.4.1.11129.2.4.3`
   *  Specify the Signed Certificate Timestamp (SCT) from the Certificate Transparency log with OID `1.3.6.1.4.1.11129.2.4.2`

An issued certificate SHOULD NOT:

   * Use a different public key scheme (ECDSA vs RSA) than its parent certificate, as some clients do not support this
   * Specify a public key that is stronger than its parent certificate. As weaknesses in keys are found, an issued certificate should be weakened before its parent, since once the parent key is compromised, it can issue new certificates.

An issued certificate MAY:

   * Specify Basic Constraints to `CA:FALSE`
   * Specify values from the OpenID Connect identity token in OIDs prefixed with `1.3.6.1.4.1.57264.1`, such as values from a CI workflow
   * Specify multiple SCTs with OID `1.3.6.1.4.1.11129.2.4.2`, denoting that the certificate has been appended to multiple logs
   * Specify the Signed Certificate Timestamp (SCT) in a response header `SCT` instead of embedding the SCT in the certificate

#### 7.3.1. CI workflows

See [https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md) for information about required claims in the CI workflow identity tokens and the corresponding X.509 extensions.

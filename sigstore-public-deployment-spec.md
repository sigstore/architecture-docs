# Sigstore Public Deployment


This document describes the technical and policy decisions for the public
deployment of Sigstore, specifically focusing on the Fulcio and Rekor deployment
for the public good instance. The [Spec: Fulcio](./fulcio-spec.md) and
[Spec: Rekor](./rekor-spec.md) documents leave many implementation choices, such as
authentication and log entry formats, to the discretion of implementers. This
document describes the specific implementation choices made for Sigstore's
public deployment that go beyond the requirements in the specification.
Additionally, this document details the use of TUF for distributing roots of
trust, and includes links to deployment respositories and resources.

## 1. Introduction

Sigstore provides authenticated metadata about digital artifacts (e.g., attestations about container images, or signatures over executable binaries) tied to digital identities (including public keys and accounts on third-party websites) while enabling public inspection (a signer may monitor all signatures created by their public key or identity, or a verifier may monitor all signatures on a particular artifact or software package).

The Sigstore specifications ([Sigstore Architecture Landing
Page](https://docs.google.com/document/u/0/d/1-OccxmZwkZZItrfOnO3RP8gku6nRbtJpth1mSW3U1Cc/edit))
describe both a general system and a specific implementation and deployment. The
general system includes an identity service ([Spec: Fulcio](./fulcio-spec.md)),
which issues short-lived certificates binding user-submitted public keys to
identities, and a transparency service ([Spec: Rekor](./rekor-spec.md)), which
records signatures and metadata in a public log.

By using these components together, a signer can tie an identity to an artifact signature via a short-lived certificate. The entry in the transparency service serves as a timestamp, ensuring that the signature was published during the validity period of the corresponding certificate.

There are Sigstore implementations of the identity service ([Fulcio](https://github.com/sigstore/fulcio)) and transparency service ([Rekor](https://github.com/sigstore/rekor)), both of which are deployed in a public good instance. [Cosign](https://github.com/sigstore/cosign), a Sigstore client implementation specifically for signing container images and other artifacts on [OCI-compatible](https://github.com/opencontainers/distribution-spec/blob/main/spec.md) container registries, uses the public good instance by default.

This document describes choices made in these implementations and this deployment above and beyond the requirements in the specifications.

### 1.1 Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 \[[RFC2119](https://www.rfc-editor.org/info/rfc2119)\] \[[RFC8174](https://www.rfc-editor.org/info/rfc8174)\] when, and only when, they appear in all capitals, as shown here.

## 2. Fulcio

Fulcio implements a certificate authority for issuing code signing certificates for a given OpenID Connect (OIDC) identity. There is a public good deployment of Fulcio run by the [OpenSSF](https://openssf.org/) and contributing organizations at [https://fulcio.sigstore.dev/](https://fulcio.sigstore.dev/).

### 2.1 Code-signing Certificates

Fulcio embeds information about the identity of a requester into the SubjectAlternativeName, Issuer, and extensions of a [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280)\-compliant [X.509v3](https://www.itu.int/rec/T-REC-X.509) certificate. The certificates are signed by an intermediate certificate generated from a [GCP Key Management Service](https://cloud.google.com/kms/docs/) key and the root certificate authority is hosted via [GCP Certificate Authority Service](https://cloud.google.com/certificate-authority-service/). Both the intermediate certificate and root certificate are distributed via TUF implemented in the [sigstore/root-signing repository](https://github.com/sigstore/root-signing).

These certificates have a validity period of 10 minutes, beginning at the time of issuance.

* [Fulcio certification specification](https://github.com/sigstore/fulcio/blob/main/docs/certificate-specification.md)

### 2.2 Authentication

Fulcio issues [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280)\-compliant [X.509v3](https://www.itu.int/rec/T-REC-X.509) certificates encoding identity information. It must authenticate the identities which it encodes into these certificates. For additional information, see [OIDC Usage in Fulcio](https://github.com/sigstore/fulcio/blob/main/docs/oidc.md).

#### 2.2.1 OpenID Connect

[OpenID Connect](https://openid.net/connect/) (OIDC) is an open identity attestation and verification standard built on top of [OAuth 2.0](https://oauth.net/2/). Sigstore uses OIDC because it is a widely adopted, industry-standard protocol that simplifies authentication, supporting both human and machine identities. Many users already have accounts with existing OIDC providers. Additionally, OIDC makes it straightforward to implement infrastructure capable of generating OIDC-compliant tokens without requiring support for the entire authentication flow. This approach ensures a seamless and secure user experience while reducing operational overhead for implementers and users.

In Fulcio, clients may authenticate using OIDC. Subject-related claims from the OIDC token are extracted and included in issued certificates.

Learn more about certificates generated from OIDC providers at [OIDC Usage in Fulcio](https://github.com/sigstore/fulcio/blob/main/docs/oidc.md).

##### 2.2.1.1 Requirements on Identity Providers

Adding a new Identity Provider (IDP) option to Fulcio helps drive adoption of Sigstore. Because identity is a critical component of the system, it is that new IDPs meet the minimum set of requirements to ensure the security and reliability of the ecosystem and users. IDPs are categorized into two types:

* Email-based OIDC Providers: These utilize the user's email or machine identity for service accounts as the certificate's subject.

* Workflow-based OIDC Providers: Designed for systems like CI/CD pipelines (e.g., GitHub Actions, GitLab CI), these require more extensive onboarding.

This document describes the minimum requirements for integrating new IDPs for either of these types: [New IDP Requirements](https://github.com/sigstore/fulcio/blob/main/docs/new-idp-requirements.md)

##### 2.2.1.2 Current Identity Providers

Sigstore runs a federated OIDC identity provider, [Dex](https://dexidp.io/). Sigstore uses Dex as an identity broker to streamline authentication across diverse identity providers, protect against and incompatibilities, and ensure a consistent experience. Additionally, Dex provides the ability to control the lifetime of tokens, ensuring tokens expire within appropriate timeframes, reducing risks associated with long-lived credentials. Users authenticate to their preferred identity provider and Dex creates an OIDC token with claims from the original OIDC token. Fulcio also supports OIDC tokens from additional configured issuers.

The Fulcio implementation allows deployers to configure which OIDC IDPs to accept. The public good instance accepts the following:

**User Authentication**

Dex:

* GitHub
* Microsoft
* Google

**Workflow Authentication**

* GitHub Actions
* GitLab CI
* BuildKite
* CodeFresh
* SPIFFE-based OIDC providers

See the [Fulcio OIDC documentation](https://github.com/sigstore/fulcio/blob/main/docs/oidc.md) for additional details.

## 3. Rekor

Rekor implements a transparency service. There is a public good deployment of Rekor run by the [OpenSSF](https://openssf.org/) and contributing organizations at [https://rekor.sigstore.dev/](https://rekor.sigstore.dev/).

### 3.1 Pluggable Types

The transparency service has what is termed a ‘pluggable type’ system. A pluggable type is a custom schema for entries stored in the transparency log. Schemas can be in multiple formats (json|yaml|xml).

The current list of supported types can be found in the [Rekor project](https://github.com/sigstore/rekor/tree/main/pkg/types). Information about adding new pluggable types can be found in the [Rekor documentation.](https://docs.sigstore.dev/logging/pluggable-types/)

See the transparency service ([Spec: Rekor](./rekor-spec.md)) document for additional information.

### 3.2 Transparency Log

Rekor is backed by a transparency log, inspired by the one in Certificate
Transparency ([RFC 6962](https://datatracker.ietf.org/doc/rfc6962/)). The
primary deviations are described in [Spec: Rekor](./rekor-spec.md); however,
like RFC 6962, the transparency service specification leaves specific parameter
choices to implementers.

* Base URL: [https://rekor.sigstore.dev/](https://rekor.sigstore.dev/)
* Hash Algorithm: SHA-256 ([RFC 6234](https://datatracker.ietf.org/doc/rfc6234/); OID 2.16.840.1.101.3.4.2.1)
* Signature Algorithm:  ECDSA (NIST P-256).
* Public Key: See [root-signing repo](https://github.com/sigstore/root-signing)
* Log ID: the SHA-256 hash of the log's public key
* Maximum Merge Delay: Rekor only returns an index after the merge is complete. Rekor does not support batching. Rekor returns an inclusion proof after waiting for an entry to be included in the log, which is expected to take <1s.
* Identity monitoring: [Rekor monitor](https://github.com/sigstore/rekor-monitor)

### 3.3 Sharding

The Certificate Transparency Log database used by Rekor currently shards every year at a minimum. The sharding is an additive in a sense that old shards will be still available at:  `https://ctfe.sigstore.dev/<SHARD>`. For example, if sharding in 2022, the old shard is available at: [https://ctfe.sigstore.dev/2022/ct/v1/get-sth](https://ctfe.sigstore.dev/2022/ct/v1/get-sth)

The convention for naming shards is that it will contain the year, followed by the instance. For example, the first shard of the year 2023 should be named 2023 and if other shards are created they will be called 2023-2, 2023-3, and so forth.

This document outlines the steps taken to shard the Rekor log: [Sharding Rekor](https://docs.sigstore.dev/logging/sharding/)

### 3.4 Timestamp Authority

Sigstore does not operate a timestamp authority at this time. We do include trusted timestamp authorities in Sigstore's TUF root. Signed timestamps can also be obtained through Rekor's SignedEntryTimestamps.

## 4. Distributing Roots-of-Trust

A client using a particular Sigstore instance needs key material for the certificate transparency log, the transparency service log, and the identity service root certificate; it may also want additional key material for verifying updates to the client itself or other, per-instance purposes.

These keys can change over time due to scheduled key rotations, log sharding, or compromise. Rather than tying a specific release of a client to a specific set of keys (which would require an upgrade of the client in order to trust new signatures), Sigstore distributes these keys using [The Update Framework](https://theupdateframework.io/) (TUF).

The resources for the Sigstore Roots-of-Trust are managed in the [sigstore/root-signing repository](https://github.com/sigstore/root-signing). To increase trust, the TUF repository is managed in a transparent and open way, allowing community members to inspect and verify all steps and updates This helps to detect any malicious activity, and ensures that users can trust the repository. When updates to the TUF repository are made, Sigstore community members are encouraged to verify all the steps before publishing a new version.

### 4.1. The Update Framework

[The Update Framework](https://theupdateframework.io/) (TUF) is a framework designed for software update systems, protecting against a number of subtle attacks (such as rollback attacks, in which an attacker serves users a formerly-valid artifact). Its [specification](https://theupdateframework.github.io/specification/latest/) is general enough for arbitrary digital artifacts. Sigstore uses TUF to distribute the public keys and certificate chains for its services, rather than a web PKI-inspired solution, because TUF offers better security against specific attacks on the distribution of key material,vastly simplifies revocations, and is feasible at the scale of a repository of key materials.

In Sigstore’s use of TUF, clients ship with a TUF trust root (which may be rotated over time). From the TUF trust root, clients can derive trust in the public keys of the Sigstore instance (the full details follow the TUF [specification](https://theupdateframework.github.io/specification/latest/) and are out-of-scope for this document).

#### 4.1.1 Artifacts

* Public keys for the certificate transparency logs, e.g. Ctfe.pub
* Certificate chains for the CA (Fulcio), e.g. Fulcio\_v1.crt.pem
* Public keys for the signature transparency log, e.g. rekor.pub
* Public key used to sign the release artifacts: artifact.pub

As keys are rotated over time, older keys have to be provided so clients can verify trust in artifacts produced in the past. Due to this multiple “versions” of the same key material are provided via the TUF repository. By relying on metadata provided, clients can find the correct key material to use for a given artifact.

Transparency logs are subject to sharding, usually once per year to keep the log fairly small. From a verifier, this may be seen as a regular key rotation and the metadata described in the previous paragraph is used to identify the correct key. Clients may also derive the [log ID from the key](https://datatracker.ietf.org/doc/html/rfc6962#section-3.2), and rely on that information when correlating a key to a log.

#### 4.1.2. Roles and Thresholds

* Root
  * 5 HSM (Yubikey) keys are held by trusted community individuals across companies, academia, and geolocations.
  * Root keys MUST be stored offline, and not used day-to-day.
  * Root signing events are expected to occur about every 4-5 months.
* Targets
  * Secured by the same HSM keys as the root to minimize the number of offline keysets.
* Snapshot
  * A GCP KMS snapshotting key.
  * The snapshot ensures consistency of the metadata files. It has a lifetime of 3 weeks and is re-signed by a GitHub workflow.
* Timestamp
  * A GCP KMS timestamping key.
  * The timestamp indicates the freshness of the metadata files. It has a lifetime of 1 week and is re-signed by two GitHub workflows.

See [sigstore/root-signing repository](https://github.com/sigstore/root-signing) for additional implementation details.

#### 4.1.3 Client

See [Sigstore TUF Client](https://docs.google.com/document/d/1QWBvpwYxOy9njAmd8vpizNQpPti9rd5ugVhji0r3T4c/edit)

## 5. Public Good Instance

The Sigstore project maintains a public good instance which consists of [Rekor](https://rekor.sigstore.dev/) and [Fulcio](https://fulcio.sigstore.dev/) deployments along with their dependencies. The resources are currently deployed to [Google Cloud Platform (GCP)](https://console.cloud.google.com/getting-started) with the configurations primarily managed by industry standard tools [Terraform](https://www.terraform.io/) and [Kubernetes Helm Charts](https://helm.sh/docs/topics/charts/).

The public good instance is maintained and monitored by volunteer engineers from several vendors as part of the [Sigstore Operations Special Interest Group (SIG)](https://github.com/sigstore/sig-public-good-operations).

### 5.1. Repositories and Resources

The Sigstore project provides the resources necessary to deploy private Sigstore infrastructure. The public good instance configuration is maintained in a private repository only available to on-call engineers but the primary resources are in the following repositories:

* [sigstore/helm-charts](https://github.com/sigstore/helm-charts)
* [sigstore/scaffolding](https://github.com/sigstore/scaffolding)
* [sigstore/sigstore-probers](https://github.com/sigstore/sigstore-probers)
* [sigstore/policy-controller](https://github.com/sigstore/policy-controller)

### 5.2 Supported Algorithms

The Sigstore public good instance supports a subset of the algorithms defined
in the [Algorithm Registry](./algorithm-registry.md). Clients that interoperate
with the public good instance **MUST** support
these algorithms in their respective contexts.

#### 5.2.1 TUF

The public good instance uses `ecdsa-sha2-256-nistp256` for all TUF signing keys.

#### 5.2.2 Fulcio

The public good instance uses `ecdsa-sha2-384-nistp384` for Fulcio's
certificate chain and `ecdsa-sha2-256-nistp256` for Fulcio's certificate
transparency log.

Clients may submit Certificate Signing Requests (CSRs) with the following
algorithms:

| Algorithm                    | Required? | Recommended?  |
|------------------------------|-----------|---------------|
| `ecdsa-sha2-256-nistp256`    | Yes       | Yes           |
| `ecdsa-sha2-384-nistp384`    | No        | Yes           |
| `ecdsa-sha2-512-nistp521`    | No        | Yes           |
| `rsa-sign-pkcs1-2048-sha256` | No        | No            |
| `rsa-sign-pkcs1-3072-sha256` | No        | No            |
| `rsa-sign-pkcs1-4096-sha256` | No        | No            |
| `ed25519`                    | No        | Yes           |

#### 5.2.3 Rekor

The public good instance may use any of the following for Rekor's
public key and signatures:

* `ecdsa-sha2-256-nistp256` (Rekor v1)
* `ecdsa-sha2-384-nistp384` (Rekor v1)
* `ed25519` (beginning with Rekor v2)

#### 5.2.4 Timestamp Authority

The public good instance uses `ecdsa-sha2-384-nistp384` for the
Timestamp Authority's certificate chain.

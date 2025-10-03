# Sigstore Public Deployment


This document describes the technical and policy decisions for the public
deployment of Sigstore, specifically focusing on the Fulcio and Rekor deployment
for the public good instance. The individual service specification documents (e.g. [Spec: Fulcio](./fulcio-spec.md),
[Spec: Rekor v2](./rekor-v2-spec.md)) leave many implementation choices, such as
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
general system includes
* an identity service ([Spec: Fulcio](./fulcio-spec.md)), which issues short-lived certificates binding user-submitted public keys to
identities
* a transparency service ([Spec: Rekor v2](./rekor-v2-spec.md), [Spec: Rekor v1](./rekor-spec.md)), which records signatures and metadata in a public log
* a timestamping authority ([IETF RFC 3161](https://datatracker.ietf.org/doc/html/rfc3161)) to prove the log entry time

By using these components together, a signer can tie an identity to an artifact signature via a short-lived certificate. The log entry combined with a timestamp ensures that the signature was published during the validity period of the certificate.

There are Sigstore implementations of the identity service ([Fulcio](https://github.com/sigstore/fulcio)),  transparency service ([Rekor v2](https://github.com/sigstore/rekor-tiles), [Rekor v1](https://github.com/sigstore/rekor)) and [Timestamp Authority](https://github.com/sigstore/timestamp-authority), all of which are deployed in a public good instance. [Cosign](https://github.com/sigstore/cosign), a Sigstore client implementation specifically for signing container images and other artifacts on [OCI-compatible](https://github.com/opencontainers/distribution-spec/blob/main/spec.md) container registries, uses the public good instance by default.

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

Rekor implements a transparency service. There are one or more public good deployments of Rekor run by the [OpenSSF](https://openssf.org/) and contributing organizations: the specific endpoints change over time and are made available in the Root-of-Trust (see [Distributing Roots-of-Trust](#distributing-roots-of-trust)).

The current Rekor implementation is version 2 and is maintained at [rekor-tiles](https://github.com/sigstore/rekor-tiles. The legacy version 1 (maintained at [rekor](https://github.com/sigstore/rekor) is still supported and is made available as an alternative in the public good instance.

### 3.1 Entry Types

Rekor v2 supports [HashedRekord v0.0.2](https://github.com/sigstore/rekor-tiles/blob/main/api/proto/rekor/v2/hashedrekord.proto)
type for basic artifact signing and [DSSE v0.0.2](https://github.com/sigstore/rekor-tiles/blob/main/api/proto/rekor/v2/dsse.proto)
type for attestations. See the transparency service ([Spec: Rekor](./rekor-v2-spec.md)) document for additional information.

Rekor v1 supports "pluggable types", see [Spec: Rekor](./rekor-spec.md).

### 3.2 Transparency Log

Rekor is backed by a transparency log, inspired by the one in Certificate
Transparency ([RFC 6962](https://datatracker.ietf.org/doc/rfc6962/)). The
primary deviations are described in [Spec: Rekor](./rekor-spec.md); however,
like RFC 6962, the transparency service specification leaves specific parameter
choices to implementers.

* Base URL: defined in [Root-of-Trust](#distributing-roots-of-trust)
* Public Key: defined in [Root-of-Trust](#distributing-roots-of-trust)
* Merkle Tree Hash Algorithm: SHA-256 ([RFC 6234](https://datatracker.ietf.org/doc/rfc6234/); OID 2.16.840.1.101.3.4.2.1)
* Signature Algorithm:
  * Rekor v1: ECDSA (NIST P-256)
  * Rekor v2: Ed25519
* Log identity
  * Rekor v1: Log Id is the SHA-256 hash of the log's public key
  * Rekor v2: Checkpoint Key Id is defined in [C2SP spec](https://github.com/C2SP/C2SP/blob/main/signed-note.md#signatures)
* Identity monitoring: [Rekor monitor](https://github.com/sigstore/rekor-monitor)

### 3.3 Sharding

Rekor and Fulcio's Certificate Transparency log currently shard every year at a minimum. Sharding means making the current log instance read-only and creating a new log instance that is writable: this keeps individual log sizes reasonable. The sharding is an additive in a sense that old shards will be still available for reading.

The convention for naming shards is that the name will contain the year, followed by optional instance number. Examples: https://ctfe.sigstore.dev/2022, https://log2025-1.rekor.sigstore.dev/.

This document outlines the steps taken to shard the Rekor log: [Sharding Rekor](https://docs.sigstore.dev/logging/sharding/).
Note that with Rekor v2 the shards are not abstracted behind a single URL so the [Root-of-Trust(#distributing-roots-of-trust) mechanism must be used to discover rekor shard URLs.

### 3.4 Timestamp Authority

Sigstore public good deployment includes a Timestamp Authority at [timestamp.sigstore.dev](https://timestamp.sigstore.dev/) but using alternative or additional Timestamp Authorities is also possible.

## 4. Distributing Roots-of-Trust

A client using a particular Sigstore instance needs key material for the certificate transparency log, the transparency service log, the identity service root certificate and timestamping authority. It will also need the URLs to the various services and other metadata such as the validity times of the services. The keys, URLs and other metadata is packaged for clients in two file formats, `trusted_root` (for verifying client) and `signing_config` (for signing client): see [sigstore-protobuf-specs](https://github.com/sigstore/protobuf-specs/blob/main/protos/sigstore_trustroot.proto) for details.

The root-of-trust data can change over time due to scheduled key rotations, log sharding, or compromise. Rather than tying a specific release of a client to a specific set of keys (which would require an upgrade of the client in order to trust new signatures), Sigstore distributes it using [The Update Framework](https://theupdateframework.io/) (TUF).

The resources for the Sigstore Roots-of-Trust are managed in the [sigstore/root-signing repository](https://github.com/sigstore/root-signing). To increase trust, the TUF repository is managed in a transparent and open way, allowing community members to inspect and verify all steps and updates This helps to detect any malicious activity, and ensures that users can trust the repository. When updates to the TUF repository are made, Sigstore community members are encouraged to verify all the steps before publishing a new version.

### 4.1. The Update Framework

[The Update Framework](https://theupdateframework.io/) (TUF) is a framework designed for software update systems, protecting against a number of subtle attacks (such as rollback attacks, in which an attacker serves users a formerly-valid artifact). Its [specification](https://theupdateframework.github.io/specification/latest/) is general enough for arbitrary digital artifacts. Sigstore uses TUF to distribute the public keys and certificate chains for its services, rather than a web PKI-inspired solution, because TUF offers better security against specific attacks on the distribution of key material,vastly simplifies revocations, and is feasible at the scale of a repository of key materials.

In Sigstore’s use of TUF, clients ship with an initial TUF root (which may be rotated over time). This allows the client to securely download current
root-of-trust files from the Sigstore instances TUF repository (the full details follow the TUF [specification](https://theupdateframework.github.io/specification/latest/) and are out-of-scope for this document).

#### 4.1.1 Artifacts in the TUF repository

Both trusted root and signing config may have multiple versions available as artifacts. Clients can select the version of trusted_root and signing_config that they support by downloading a specific version from the TUF repository. New versions should be rare, but clients should switch to a newer version when one is made available.
* `trusted_root.json` (in future `trusted_root.v<MAJOR>.<MINOR>.json`)
* `signing_config.v<MAJOR>.<MINOR>.json`
* All other artifacts should be considered deprecated

The artifact contents and the version numbers are defined in the protobufs in [sigstore-protobuf-specs](https://github.com/sigstore/protobuf-specs/blob/main/protos/sigstore_trustroot.proto).


#### 4.1.2. Roles and Thresholds

* Root
  * 5 HSM (Yubikey) keys are held by trusted community individuals across companies, academia, and geolocations.
  * Root keys MUST be stored offline, and not used day-to-day.
  * Root signing events are expected to occur about every 4-5 months.
* Targets
  * Secured by the same HSM keys as the root to minimize the number of offline keysets.
* Snapshot & timestamp
  * Snapshot and timestamp roles ensures consistency and freshness of the artifacts
  * A new timestamp is signed daily by a GCP KMS key: a timestamp older than a week will not be accepted by clients

See [sigstore/root-signing repository](https://github.com/sigstore/root-signing) for additional implementation details.

#### 4.1.3 Client

See [Sigstore TUF Client](https://docs.google.com/document/d/1QWBvpwYxOy9njAmd8vpizNQpPti9rd5ugVhji0r3T4c/edit). Note that the document is
outdated with regards to artifact discovery: clients should only look for the trusted root and signing config files they have decided to support.

## 5. Public Good Instance

The Sigstore project maintains a public good instance which consists of
* a [Fulcio](https://fulcio.sigstore.dev/) deployment
* a [Timestamp Authority](https://timestamp.sigstore.dev/) deployment
* a [federating OIDC provider](https://oauth2.sigstage.dev/) (dex) deployment
* a [Certificate Transparency log](https://ctfe.sigstore.dev/2022) deployment
* Multiple Rekor deployments: Rekor v1 at https://rekor.sigstore.dev/ and Rekor v2 deployments at subdomains of rekor.sigstore.dev

While most services have known endpoint URLs, the correct current endpoints are always available via [Root-of-Trust](#distributing-roots-of-trust).

The resources are currently deployed to [Google Cloud Platform (GCP)](https://console.cloud.google.com/getting-started) with the configurations primarily managed by industry standard tools [Terraform](https://www.terraform.io/) and [Kubernetes Helm Charts](https://helm.sh/docs/topics/charts/).

The public good instance is maintained and monitored by volunteer engineers from several vendors as part of the [Sigstore Operations Special Interest Group (SIG)](https://github.com/sigstore/sig-public-good-operations).

### 5.1. Repositories and Resources

The Sigstore project provides the resources necessary to deploy private Sigstore infrastructure. The public good instance configuration is maintained in a private repository only available to on-call engineers but the primary resources are in the following repositories:

* [sigstore/helm-charts](https://github.com/sigstore/helm-charts)
* [sigstore/terraform-modules](https://github.com/sigstore/terraform-modules)
* [sigstore/sigstore-probers](https://github.com/sigstore/sigstore-probers)
* [sigstore/policy-controller](https://github.com/sigstore/policy-controller)

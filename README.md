# Sigstore Architecture Documentation

The purpose of this repository is to store a community-edited, formal description of the architecture of Sigstore. 

   * [Sigstore Client Spec](client-spec.md) - This document specifies an architecture for using an automated certificate authority specifically, timestamping service, and transparency service for signing digital payloads.
   * [Fulcio, A Certificate Authority for Code Signing](fulcio-spec.md) - This document describes Fulcio, a certificate authority for issuing short-lived code signing certificates for an OpenID Connect (OIDC) identity, such as an email address. 
   * [Rekor, A Transparency Service](rekor-spec.md) - This document describes Rekor, a signature tranparency service that securely records and makes verifiable the metadata of signed software artifacts, ensuring trust and integrity in the software supply chain.
      * [Rekor 2.0 Proposal](https://docs.google.com/document/d/1Mi9OhzrucIyt-UCLk_FxO2_xSQZW9ow9U3Lv0ZB_PpM/edit?resourcekey=0-4rPbZPyCS7QDj26Hk0UyvA&tab=t.0#heading=h.bjitqo6lwsmn) - ⚠️ Sigstore is moving towards a new design for Rekor, based on a [tile-based log](https://transparency.dev/articles/tile-based-logs/) to simplify maintenance, lower operational costs and improve scalability. This change is imminent and a spec doc will be made available in this repo in due course once the community makes the transistion. (To access the proposal doc you must be a member of the [sigstore-dev@ Google group](https://groups.google.com/g/sigstore-dev))    
   * [Sigstore Public Deployment](sigstore-public-deployment-spec.md) - This document describes the technical and policy decisions for the public deployment of Sigstore, specifically focusing on the Fulcio and Rekor deployment for the public good instance. This document details the specific implementation choices made for Sigstore's public deployment that go beyond the requirements in the specification. Additionally, this document details the use of TUF for distributing roots of trust, and includes links to deployment respositories and resources.

## Goals
The goals of these architecture documents are:

   1. **Enable Interoperability Across Sigstore Client Implementations** The client specification aims to make it easy to develop Sigstore clients that are compatible across various languagues (e.g. Go, Python, Rust, Ruby, Java) and at different stages of maturity. With the provided specification document, developers can implement a client, knowing it will interoperate with other clients and Sigstore implementations. Additionally, the specifications will clarify which features are mandatory for compliance and which are optional enhancements. This clarity supports consistent conformance testing and auditing for various implementations.
   2. **Ensure Stability for Sigstore’s Public Deployment** A well-defined specification assures users and developers that Sigstore is reliable and stable. To support this, the specifications will:
      * Define requirements for stability and backward compatibility.
      * Enable controlled updates through versioning guarantees and a structured change-management process.
      * Establish clear expectations around system reliability and support
   3. **Describe the Current State of Sigstore** These documents aim to describe in detail the architecture of the Sigstore building blocks, as well as description of how they fit together to enable its use case and encourage broader adoption. While improvements to Sigstore are on the horizon, these specifications will focus on current functionality. They are, however, living documents and will be updated as Sigstore evolves. Imminent changes may be noted, while speculative changes will generally be omitted.
   4. **Work Toward Formalizing the Specification** This repository serves as a collaborative, community-driven description of Sigstore's architecture, with an emphasis on deriving specification from working code. The ultimate goal is provide robust, comprehensive, standardized architecture documents suitable for submission to a standards body, such as the IETF, in the future. By grounding the specifications in practical implementations, this approach ensures real-world applicability and supports broader adoption and alignment across the community and industry. This repository was forked from [https://github.com/martinthomson/i-d-template/](https://github.com/martinthomson/i-d-template/), which provides many features to help in publishing.

## Development

Feedback and improvements are welcome. To participate, simply open an issue or suggest edits to the docs through a pull request. Before making big changes, it's probably prudent to check in on the `#architecture-docs` channel in the [sigstore slack](https://sigstore.slack.com/) (invitation link [here](https://links.sigstore.dev/slack-invite)).

The original architecture documents (now archived) from which these specs are derived can be accessed here: [Landing page](https://docs.google.com/document/d/1-OccxmZwkZZItrfOnO3RP8gku6nRbtJpth1mSW3U1Cc/edit) for details (you must be a member of the [sigstore-dev@ Google group](https://groups.google.com/g/sigstore-dev) to access). 

## License

Please note that since sigstore is an OpenSSF affiliated project, all specifications are published under the [Community Specification model and license](https://github.com/CommunitySpecification/1.0) from the [Joint Development Foundation](https://www.jointdevelopment.org).

All relevant terms can be found in the [governance](https://github.com/sigstore/architecture-docs/tree/main/governance) subdirectory of this repository.

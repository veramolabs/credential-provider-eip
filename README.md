---
eip: xyz
title: Add Credential Provider related methods to the JSON-RPC
author: Oliver Terbu (@awoie)
discussions-to: https://github.com/ethereum/EIPs/issues/TBD
status: Draft
type: Standards Track
category: Interface
created: 2021-08-18
---

<!--You can leave these HTML comments in your merged EIP and delete the visible duplicate text guides, they will not appear and may be helpful to refer to if you edit it again. This is the suggested template for new EIPs. Note that an EIP number will be assigned by an editor. When opening a pull request to submit your EIP, please use an abbreviated title in the filename, `eip-draft_title_abbrev.md`. The title should be 44 characters or less.-->

## Simple Summary
<!--"If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the EIP.-->
Add new methods to the JSON-RPC for storing, creating, selectively disclosing and proving control of offchain- and onchain credentials under a new `creds_*` prefix.

## Abstract
<!--A short (~200 word) description of the technical issue being addressed.-->

This EIP describes three methods to add to the JSON-RPC that enables wallets to act as a Credential Provider (CP) to support *Verifiabe Credentials* (VCs) storage, issuance, selective disclosure and proof of control. VCs are usually self-certifyable attestations from an issuer about the owner of the VC encoded in the credential subject. The owner of the VC can selective disclose information from those VCs and prove control of the VC to a third-party. Please visit https://www.w3.org/TR/vc-data-model/ for a full explaination of VCs. Since the the VC data model is very flexible, this EIP enforces specific rules on VCs and supported proof types to facilitate developer experience and interoperability. This is important for use cases such as sign-in, sign-up and decentralized reputation-based authorization.

This EIP is complementary to EIP-2844.

## Motivation
<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

Web3 is missing a coherent method for requesting a login assertions for sign-in and sign-up. The majority of Web3 projects are using an approach where they cryptographically bind a signature produced by the wallet to the identity assertion through either [personal_sign]() or [EIP-712](). The identity assertion becomes self-certifiable with this approach. The identifier for the user is either the Ethereum address or the ENS name depending on availability. To improve privacy it is important to introduce a mechanism that allows people to selective disclose the linkage between their primary identifier and their Ethereum account. This can be done through VCs and DIDs. Web3 applications like DAOs, Defi, NFT market places etc. need verifiable offchain and onchain reputation to enable certain features for their end users. Using VCs with LD-Proofs, it will be possible to find a standard representation for all types of identity assertions, and specifically with LD-Proofs, those identity attestations can contain proofs that can be consumed in offchain and onchain. [EIP-2844]() is a good starting point but the solves the problem only partially. This EIP proposes to build on top of [EIP-2844]() and and introduce new JSON-RPC methods that are needed to build decentralized reputation for offchain and onchain use.

<!-- example copied from eip2844
There has been one main previous effort ([#130](https://github.com/ethereum/EIPs/issues/130), [#1098](https://github.com/ethereum/EIPs/pull/1098)) to add decryption to Ethereum wallets in a standard way. This previous approach used a non standard way to encode and represent data encrypted using `x25519-xsalsa20-poly1305`. While this approach does provide a functional way to add encryption support to wallets, it does not take into account similar work that has gone into standardizing the way encrypted data is represented, namely using [JOSE](https://datatracker.ietf.org/wg/jose/documents/) and [COSE](https://datatracker.ietf.org/wg/cose/documents/). Both of these are standards from IETF for representing signed and encrypted objects. Another shortcoming of the previous approach is that it's impossible to retrieve the `x25519` public key from another user if only an Ethereum address is known. Public key discoverability is at the core of the work that is happening with the [W3C DID standard](https://w3c.github.io/did-core), where given a DID a document which contains public keys can always be discovered. Implementations of this standard already exist and are adopted within the Ethereum community, e.g. [`did:ethr`](https://github.com/decentralized-identity/ethr-did-resolver/) and [`did:3`](https://github.com/3box/3id-resolver). Interoperability between JOSE/COSE and DIDs [already exists](https://github.com/decentralized-identity/did-jwt), and work is being done to [strengthen it](https://github.com/decentralized-identity/did-jose-extensions). Adding support for JOSE/COSE and DIDs will enable Ethereum wallets to support a wide range of new use cases such as more traditional authentication using JWTs, as well as new emerging technologies such as [Secure Data Stores](https://identity.foundation/secure-data-store/) and [encrypted data in IPFS](https://github.com/ipld/specs/pull/269).
-->

## Specification
Three new JSON-RPC methods are specified under the new `creds_*` prefix.

### Supported LD-Proofs

This is a registry of supported LD-Proofs by this specification. New LD-proof types MAY be added through PRs. The 
MUST contain a link to the LD-Proof specification as well as a decription why support is  needed. As a rule of thumb, 
new LD-Proof types MUST be registered in the W3C CCG [LD-Proof Suite Registry](https://w3c-ccg.github.io/ld-cryptosuite-registry/).

Credential Providers MUST support the following LD-Proof types:
- [`EthereumEip712Signature2021`](TBD)
- [`JsonWebSignature2020`](https://github.com/w3c-ccg/lds-jws2020), only Ed25519 and secp256k1


### Supported Verifiable Credentials Profile

VCs that can be used with this specification MUST be valid JSON-LD and MUST contain a valid LD-Proof in the `proof` property. JWT-based VCs and proofs are intentionally not supported.

### Store

Stores the given VC in the CP.

##### Method: 

`creds_store`

##### Params:

* `vc` - A Verifiable Credential.

##### Returns:

* `error` - If `vc` was malformed or does not comply with the Verifialbe Credentials Profile defined in this specification.

### Issue

Issues a VC with the given payload using one of the CP's DIDs.

##### Method: 

`creds_issue`

##### Params:

* `payload` - The payload of the Verifiable Credential to be issued.
* `verificationMethod` - A DID URI that indicates which method (e.g., key) to use to create the proof over the `payload`, e.g., `did:example:0xaaaa#my-secp256k1-key`

##### Returns:

* `vc` - A Verifiable Credential that was issued by the CP. 
* `error` - If `payload` was malformed, does not comply with the Verifialbe Credentials Profile defined in this specification, or the `verificationMethod` was not found or is not mangaged by the CP.

### Present

Selective discloses information from the CP. Optionally, holder binding can be requested. For the query, 
we will use the [DIF Presentation Exchange](https://identity.foundation/presentation-exchange/) data model. 

##### Method: 

`creds_present`

##### Params:

* `presentation_defintion` - Defines the selective disclosure query with optional holder binding.
* `domain` - OPTIONAL: if holder binding was requested, this parameter is mandatory.
* `challenge` - OPTIONAL: if holder binding was requested, this parameter is mandatory. 

##### Returns:

* `presentation_submission` - Defines where the requested information can be found in the VP.
* `vp` - A Verifiable Presentation (VP) that contains the requested disclosed VCs from the CP.
* `error` - If `presentation_definition` was malformed, does not comply with the Verifialbe Credentials Profile defined in this specification.

##### Examples:

TBD


## Rationale
<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

TBD

## Implementation
<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

- TBD
- TBD
- TBD

## Security Considerations
<!--All EIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. EIP submissions missing the "Security Considerations" section will be rejected. An EIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.-->

TBD: something on secure challenge (aka nonce)

TBD

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
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

This EIP describes three methods to add to the JSON-RPC that enables wallets to act as a Credential Provider (CP) to support _Verifiabe Credentials_ (VCs) storage, issuance, selective disclosure and proof of control. VCs are usually self-certifiable attestations from an issuer about the owner of the VC encoded in the credential subject. The owner of the VC can selectively disclose information from those VCs and prove control of the VC to a third-party. Please visit https://www.w3.org/TR/vc-data-model/ for a full explaination of VCs. Since the VC data model is very flexible, this EIP enforces specific rules on VCs and supported proof types to facilitate developer experience and interoperability. This is important for use cases such as sign-in, sign-up and decentralized reputation-based authorization.

This EIP is complementary to [EIP-2844](https://eips.ethereum.org/EIPS/eip-2844).

## Motivation

<!--The motivation is critical for EIPs that want to change the Ethereum protocol. It should clearly explain why the existing protocol specification is inadequate to address the problem that the EIP solves. EIP submissions without sufficient motivation may be rejected outright.-->

Web3 applications like DAOs, Defi, NFT market places etc. need verifiable offchain and onchain reputation to enable certain features for their end users. Using VCs with LD-Proofs, it will be possible to find a standard representation for all types of identity assertions, and specifically with LD-Proofs, those identity attestations can contain proofs that can be consumed offchain and onchain. [EIP-2844](https://eips.ethereum.org/EIPS/eip-2844) is a good starting point but solves the problem only partially. This EIP builds on top of [EIP-2844](https://eips.ethereum.org/EIPS/eip-2844) and introduces new JSON-RPC methods that are needed to build decentralized reputation for offchain and onchain use.

Web3 is missing a coherent method for requesting identity assertions from their users, e.g. for sign-in and sign-up. The majority of Web3 projects are using an approach where they cryptographically bind a signature produced by the wallet to the identity assertion through either [eth.personal.sign](https://web3js.readthedocs.io/en/v1.4.0/web3-eth-personal.html#sign) or [EIP-712](https://eips.ethereum.org/EIPS/eip-712). The identity assertion becomes self-certifiable with this approach. The identifier for the user is the Ethereum address. To improve privacy it is important to introduce a mechanism that allows people to selectively disclose the linkage between their primary identifier and their Ethereum address. This can be done through VCs and DIDs.

## Specification

Three new JSON-RPC methods are specified under the new `creds_*` prefix.

### Verifiable Credential Proofs

This section provides guidance on recommended [LD-Proof Suites](https://w3c-ccg.github.io/ld-proofs/)
and [IANA JWS algorithm](https://www.iana.org/assignments/jose/jose.xhtml) support of embedded and
external proofs for VCs.

#### Embedded Proofs

CPs SHOULD support the following LD-Proof types for embedded proofs (i.e. VC-LDP):
- [`EthereumEip712Signature2021`](https://w3id.org/security/suites/eip712sig-2021)
- [`JsonWebSignature2020`](https://w3id.org/security/suites/jws-2020), only Ed25519 and secp256k1
- [`BbsBlsSignature2020`](https://w3id.org/security/suites/bls12381-2020)
- [`BbsBlsBoundSignature2020`](https://w3id.org/security/suites/bls12381-2020)

#### External Proofs

CPs SHOULD support the following [IANA](https://www.iana.org/assignments/jose/jose.xhtml) JWS algorithms
for external proofs (i.e. VC-JWT):
- [`ES256K`](https://www.rfc-editor.org/rfc/rfc8812.html)
- [`EdDSA`](https://www.rfc-editor.org/rfc/rfc8037.html)

### Supported Verifiable Credentials Profile

VCs that can be used with this specification MUST be valid JSON-LD. VCs and VPs SHOULD use any of the proofs recommended by [Embedded Proofs](#EmbeddedProofs) or [External Proofs](#ExternalProofs).

### Store

Stores the given VC in the CP.

##### Method:

`creds_store`

##### Params:

- `vc` - A Verifiable Credential.

##### Returns:

- `error` - OPTIONAL. If `vc` was malformed or does not comply with the Verifialbe Credentials Profile defined in this specification.

### Issue

Issues a VC with the given payload using one of the CP's DIDs.

##### Method:

`creds_issue`

##### Params:

- `payload` - REQUIRED. The payload of the Verifiable Credential to be issued.
- `preferreded_proofs` - OPTIONAL. An ordered array of preferred proof formats and types for the VC to be issued. Each array item is an object with two properties, `format` and `type`. `format` indicates the preferred proof type, which is either `jwt` for (Embedded Proofs) or `ldp` for (External Proofs). The `type` refers to proof type of the VC (see [Verifiable Credentials Proofs](#VerifiableCredentialsProofs) for a list of valid combinations). If the wallet does not support any of the preferred proofs, the wallet can select a format and type from the list defined in [Verifiable Credentials Proofs](#VerifiableCredentialsProofs) of as a fallback.

##### Returns:

- `vc` - OPTIONAL. Present if the call was successful. A Verifiable Credential that was issued by the CP.
- `error` - OPTIONAL. If `payload` was malformed, does not comply with the Verifialbe Credentials Profile defined in this specification.

### Present

Selectively discloses information from the CP. Optionally, holder binding can be requested. For the query,
we will use the [DIF Presentation Exchange](https://identity.foundation/presentation-exchange/) data model.

##### Method:

`creds_present`

##### Params:

- `presentation_defintion` - Defines the selective disclosure query with optional holder binding.
- `domain` - OPTIONAL. If holder binding was requested, this parameter is mandatory.
- `challenge` - OPTIONAL. If holder binding was requested, this parameter is mandatory.

##### Returns:

- `vp` - OPTIONAL. Present if the call was successful. It contains a _Verifiable Presentation_ (VP) that contains the requested VCs from the CP.
- `error` - OPTIONAL. Present if `presentation_definition` was malformed, does not comply with the Verifialbe Credentials Profile defined in this specification.

##### Examples:

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://identity.foundation/presentation-exchange/submission/v1"
  ],
  "type": ["VerifiablePresentation", "PresentationSubmission"],
  "holder": "did:example:123",
  "presentation_submission": {
    "id": "1d257c50-454f-4c96-a273-c5368e01fe63",
    "definition_id": "32f54163-7166-48f1-93d8-ff217bdb0654",
    "descriptor_map": [
      {
        "id": "vaccination_input",
        "format": "ldp_vp",
        "path": "$.verifiableCredential[0]"
      }
    ]
  },
  "verifiableCredential": [
    {
      "@context": [
        "https://www.w3.org/2018/credentials/v1",
        "https://w3id.org/vaccination/v1",
        "https://w3id.org/security/bbs/v1"
      ],
      "id": "urn:uvci:af5vshde843jf831j128fj",
      "type": ["VaccinationCertificate", "VerifiableCredential"],
      "description": "COVID-19 Vaccination Certificate",
      "name": "COVID-19 Vaccination Certificate",
      "expirationDate": "2029-12-03T12:19:52Z",
      "issuanceDate": "2019-12-03T12:19:52Z",
      "issuer": "did:example:456",
      "credentialSubject": {
        "id": "urn:bnid:_:c14n2",
        "type": "VaccinationEvent",
        "batchNumber": "1183738569",
        "countryOfVaccination": "NZ"
      },
      "proof": {
        "type": "BbsBlsSignatureProof2020",
        "created": "2021-02-18T23:04:28Z",
        "nonce": "JNGovx4GGoi341v/YCTcZq7aLWtBtz8UhoxEeCxZFevEGzfh94WUSg8Ly/q+2jLqzzY=",
        "proofPurpose": "assertionMethod",
        "proofValue": "AB0GQA//jbDwMgaIIJeqP3fRyMYi6WDGhk0JlGJc/sk4ycuYGmyN7CbO4bA7yhIW/YQbHEkOgeMy0QM+usBgZad8x5FRePxfo4v1dSzAbJwWjx87G9F1lAIRgijlD4sYni1LhSo6svptDUmIrCAOwS2raV3G02mVejbwltMOo4+cyKcGlj9CzfjCgCuS1SqAxveDiMKGAAAAdJJF1pO6hBUGkebu/SMmiFafVdLvFgpMFUFEHTvElUQhwNSp6vxJp6Rs7pOVc9zHqAAAAAI7TJuDCf7ramzTo+syb7Njf6ExD11UKNcChaeblzegRBIkg3HoWgwR0hhd4z4D5/obSjGPKpGuD+1DoyTZhC/wqOjUZ03J1EtryZrC+y1DD14b4+khQVLgOBJ9+uvshrGDbu8+7anGezOa+qWT0FopAAAAEG6p07ghODpi8DVeDQyPwMY/iu2Lh7x3JShWniQrewY2GbsACBYOPlkNNm/qSExPRMe2X7UPpdsxpUDwqbObye4EXfAabgKd9gCmj2PNdvcOQAi5rIuJSGa4Vj7AtKoW/2vpmboPoOu4IEM1YviupomCKOzhjEuOof2/y5Adfb8JUVidWqf9Ye/HtxnzTu0HbaXL7jbwsMNn5wYfZuzpmVQgEXss2KePMSkHcfScAQNglnI90YgugHGuU+/DQcfMoA0+JviFcJy13yERAueVuzrDemzc+wJaEuNDn8UiTjAdVhLcgnHqUai+4F6ONbCfH2B3ohB3hSiGB6C7hDnEyXFOO9BijCTHrxPv3yKWNkks+3JfY28m+3NO0e2tlyH71yDX0+F6U388/bvWod/u5s3MpaCibTZEYoAc4sm4jW03HFYMmvYBuWOY6rGGOgIrXxQjx98D0macJJR7Hkh7KJhMkwvtyI4MaTPJsdJGfv8I+RFROxtRM7RcFpa4J5wF/wQnpyorqchwo6xAOKYFqCqKvI9B6Y7Da7/0iOiWsjs8a4zDiYynfYavnz6SdxCMpHLgplEQlnntqCb8C3qly2s5Ko3PGWu4M8Dlfcn4TT8YenkJDJicA91nlLaE8TJbBgsvgyT+zlTsRSXlFzQc+3KfWoODKZIZqTBaRZMft3S/",
        "verificationMethod": "did:example:123#key-1"
      }
    }
  ],
  "proof": {
    "type": "Ed25519Signature2018",
    "verificationMethod": "did:example:123#key-0",
    "created": "2021-05-14T20:16:29.565377",
    "proofPurpose": "authentication",
    "challenge": "3fa85f64-5717-4562-b3fc-2c963f66afa7",
    "jws": "eyJhbGciOiAiRWREU0EiLCAiYjY0IjogZmFsc2UsICJjcml0IjogWyJiNjQiXX0..7M9LwdJR1_SQayHIWVHF5eSSRhbVsrjQHKUrfRhRRrlbuKlggm8mm_4EI_kTPeBpalQWiGiyCb_0OWFPtn2wAQ"
  }
}
```

## Rationale

<!--The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages. The rationale may also provide evidence of consensus within the community, and should discuss important objections or concerns raised during discussion.-->

The [VC Data Model](https://www.w3.org/TR/vc-data-model/) define [Verifiable Presentation](https://www.w3.org/TR/vc-data-model/#presentations) but does not provide detail on how to express the constraints that a relying party has on a presentation. The [Presentation Exchange Data Model](https://identity.foundation/presentation-exchange/) defines a definition and submission format for among other things, verifiable presentations.

The [Universal Wallet Interop Spec](https://w3id.org/wallet) describes how to use concrete protocols such as [Wallet Connect](https://docs.walletconnect.org/) and [WACI Presentation Exchange](https://identity.foundation/waci-presentation-exchange/) with [DID Comm Messaging](https://identity.foundation/didcomm-messaging/spec/).

In cases where a holder is directly connected to a verifier over a secure transport, encryption and messaging related standards such as DIDComm are not required, however interoperable data models for expressing presentation requirements and submissions are still needed to support interoperability with existing standards.

This proposal defines a set of API extensions that would enable web3 wallet providers to offer wallet and credential interactions to web origins that already support web3 wallet providers.

This functionality is similar to the interfaces supported by the [credential handler api](https://w3c-ccg.github.io/credential-handler-api/), which does not support the [Presentation Exchange Data Model](https://identity.foundation/presentation-exchange/) specification.

## Implementation

<!--The implementations must be completed before any EIP is given status "Final", but it need not be completed before the EIP is accepted. While there is merit to the approach of reaching consensus on the specification and rationale before writing code, the principle of "rough consensus and running code" is still useful when it comes to resolving many discussions of API details.-->

- TBD
- TBD
- TBD

## Security Considerations

<!--All EIPs must contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed. EIP submissions missing the "Security Considerations" section will be rejected. An EIP cannot proceed to status "Final" without a Security Considerations discussion deemed sufficient by the reviewers.-->

User consent must be obtained prior to accessing wallet apis.

The relying party MUST ensure that the challenge required by the verifiable presentation are is sufficiently random, and used only once, or that it is some form of expiring verifiable credential encoded as a string.

TODO: The domain must match the web origin requesting or providing credentials or presentations?

In the case that no holder binding is required, the submission details can be tampered with.

TBD

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

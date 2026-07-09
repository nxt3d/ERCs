---
eip: <to be assigned>
title: Counterfactual ERC-8004 Registration
description: Counterfactual identity claims for ERC-8004 registrations.
author: Prem Makeig (@nxt3d)
discussions-to: TODO
status: Draft
type: Standards Track
category: ERC
created: 2026-05-20
requires: 712, 721, 1271, 8004
---

## Abstract

Registering AI agents onchain allows them to be discovered without taking on counterparty risk, such as being censored or deleted.

[ERC-8004](./eip-8004.md) is an AI agent registry that provides onchain registration. However, it has a number of limitations that this ERC fixes. With ERC-8004 each registration mints an [ERC-721](./eip-721.md) identity token, which is expensive to do for every agent. Also the agent identifier does not include the chain it was created on, so it is not unique across chains.

This ERC defines counterfactual ERC-8004 registration, which records an agent onchain without minting a token. It also creates a globally unique agent identifier derived deterministically from the registration data.

## Motivation

Counterfactual ERC-8004 registation provides many of the benefits of ERC-8004 registartion with a mucy lower cost. Insated of minting a new token, Counterfactual registration uses an existing NFT as the root of the identity, and ownership record. For existing NFT collections, this dramatically reduces the costs for agent registation. 

Instead of using onchain storage, for counterfactual registartion the claim is published as an onchain event that can be indexed and even updated if necessary. Later a conterfactual registration can be promoted to a onchain ERC-8004 registration if necessary. 

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

This ERC defines counterfactual ERC-8004 registration via event emission. The reason that it is called coungerfactual, is becuase all the inputs are the same for registration, and therefore the inputs `could` but instead of registering the inputs onchain, we hash the inputs and emit the event.

### Interface

Counter factual registartion is a unitque interface of ERC-8217, which is a singleton adapter contract for ERC-8004, allowing for the additional functionality. ERC-8217 adapter contacts MUST implement the following interface:

```solidity
pragma solidity ^0.8.24;

interface IERC8004CounterfactualRegistration {
    enum TokenStandard {
        ERC721,
        ERC1155,
        ERC6909, 
        ERC1155F, 
        ERC6909F
    }

    struct MetadataEntry {
        string metadataKey;
        bytes metadataValue;
    }

    /// @notice Computes the canonical counterfactual registration hash.
    function registrationHash(
        address tokenContract,
        uint256 tokenId
    ) external view returns (bytes32);

    /// @notice Counterfactual registration claim (emits event, no registry write, no SSTORE).
    function counterfactualRegister(
        TokenStandard standard,
        address tokenContract,
        uint256 tokenId,
        string calldata agentURI,
        MetadataEntry[] calldata metadata
    ) external returns (bytes32 computedHash);

    /// @notice Convenience overload equivalent to counterfactualRegister with empty metadata.
    function counterfactualRegister(
        TokenStandard standard,
        address tokenContract,
        uint256 tokenId,
        string calldata agentURI
    ) external returns (bytes32 computedHash);

    /// @notice Signature-authorized counterfactual registration. The token owner signs an EIP-712
    /// payload and ANY caller may submit it, so a registration can be bundled into a transaction the
    /// owner does not send (for example, a mint that registers the agent at mint time). Emits
    /// CounterfactualAgentRegistered (and CounterfactualAgentWalletSet when agentWallet != address(0))
    /// with emitter = owner.
    function counterfactualRegisterWithSig(
        TokenStandard standard,
        address tokenContract,
        uint256 tokenId,
        string calldata agentURI,
        MetadataEntry[] calldata metadata,
        address agentWallet,
        address owner,
        uint256 expiration,
        bytes calldata signature
    ) external returns (bytes32 computedHash);

    /// @notice Counterfactual agent URI update (emits event, no registry write, no SSTORE).
    function counterfactualSetAgentURI(
        TokenStandard standard,
        address tokenContract,
        uint256 tokenId,
        string calldata newURI
    ) external;

    /// @notice Counterfactual single metadata write (emits event, no registry write, no SSTORE).
    function counterfactualSetMetadata(
        TokenStandard standard,
        address tokenContract,
        uint256 tokenId,
        string calldata metadataKey,
        bytes calldata metadataValue
    ) external;

    /// @notice Counterfactual batch metadata write (emits event, no registry write, no SSTORE).
    function counterfactualSetMetadataBatch(
        TokenStandard standard,
        address tokenContract,
        uint256 tokenId,
        MetadataEntry[] calldata metadata
    ) external;

    /// @notice Counterfactual agent wallet assignment (emits event, no registry write, no SSTORE).
    function counterfactualSetAgentWallet(
        TokenStandard standard,
        address tokenContract,
        uint256 tokenId,
        address newWallet
    ) external;

    /// @notice Counterfactual agent wallet clear (emits event, no registry write, no SSTORE).
    function counterfactualUnsetAgentWallet(
        TokenStandard standard,
        address tokenContract,
        uint256 tokenId
    ) external;

    /// @notice Counterfactual registration claim (emitted event, no registry write, no SSTORE).
    event CounterfactualAgentRegistered(
        bytes32 indexed registrationHash,
        address indexed tokenContract,
        uint256 indexed tokenId,
        uint8 version,
        TokenStandard standard,
        string agentURI,
        MetadataEntry[] metadata,
        address emitter
    );

    /// @notice Counterfactual agent URI update (emitted event, no registry write, no SSTORE).
    event CounterfactualAgentURISet(
        bytes32 indexed registrationHash,
        address indexed tokenContract,
        uint256 indexed tokenId,
        uint8 version,
        string newURI,
        address emitter
    );

    /// @notice Counterfactual single metadata write (emitted event, no registry write, no SSTORE).
    event CounterfactualMetadataSet(
        bytes32 indexed registrationHash,
        address indexed tokenContract,
        uint256 indexed tokenId,
        uint8 version,
        string metadataKey,
        bytes metadataValue,
        address emitter
    );

    /// @notice Counterfactual batch metadata write (emitted event, no registry write, no SSTORE).
    event CounterfactualMetadataBatchSet(
        bytes32 indexed registrationHash,
        address indexed tokenContract,
        uint256 indexed tokenId,
        uint8 version,
        MetadataEntry[] metadata,
        address emitter
    );

    /// @notice Counterfactual agent wallet assignment (emitted event, no registry write, no SSTORE).
    event CounterfactualAgentWalletSet(
        bytes32 indexed registrationHash,
        address indexed tokenContract,
        uint256 indexed tokenId,
        uint8 version,
        address newWallet,
        address emitter
    );

    /// @notice Counterfactual agent wallet clear (emitted event, no registry write, no SSTORE).
    event CounterfactualAgentWalletUnset(
        bytes32 indexed registrationHash,
        address indexed tokenContract,
        uint256 indexed tokenId,
        uint8 version,
        address emitter
    );
}
```

The `TokenStandard` enum values are `ERC721 == 0`, `ERC1155 == 1`, and `ERC6909 == 2`. Implementations MUST support these standards and MAY support additional token standards. If an implementation does not support a supplied `standard`, the call MUST revert.

Every counterfactual event defined in this ERC MUST emit `version == 1`. Implementations MAY expose the current payload version through an additional pure function or constant, but consumers MUST rely on the per-event `version` field when decoding logs.

### Registration hash

For every counterfactual function, the adapter MUST compute:

```solidity
registrationHash = keccak256(abi.encode(block.chainid, address(this), tokenContract, tokenId));
```

`address(this)` is the emitting adapter address. The hash binds a counterfactual identity to one chain, one adapter, and one external token. The token standard is deliberately excluded from the hash, so a token has one identity regardless of which token interface it is registered through, and the identity is stable across payload-schema revisions. The standard is still supplied to each counterfactual function, validated at call time by the control check (see Authorization), and carried in every counterfactual event. A contract that serves the same `(tokenContract, tokenId)` under more than one token standard therefore resolves to a single identity.

`registrationHash(address tokenContract, uint256 tokenId)` MUST be deterministic and MUST NOT depend on mutable contract storage. Apart from `block.chainid` and `address(this)`, it MUST use only its function arguments.

### Event consumption

Indexers and clients MUST treat counterfactual state as state derived from onchain logs. For a given `registrationHash`, all in-protocol updates MUST be sequenced together as a single ordered stream, regardless of event type or emission mechanism. Indexers and clients MUST apply updates from this unified stream in order, and MUST treat the latest valid update for any given field or metadata key as authoritative. A `CounterfactualMetadataBatchSet` event contains independent writes for each entry in `metadata`, and all entries share the position of the enclosing event in the stream.

Indexers and clients SHOULD order events by block number, transaction index, and log index. Indexers and clients SHOULD persist the `version` of the last-applied event for each derived field. This lets a replay under a different payload version be flagged rather than silently overwrite decoded state. If a chain reorganization removes an event, indexers and clients MUST remove the derived state from that event and replay the surviving canonical log sequence.

### Authorization

For every counterfactual function, the caller MUST control the external token identified by `standard`, `tokenContract`, and `tokenId`. The adapter MUST perform this authorization check at call time, before emitting any counterfactual event.

Control means one of the following:

1. **Direct ownership.** The caller currently owns the external token at the call site. For ERC-721, ownership is read via `ownerOf(tokenId)` on `tokenContract`. For ERC-1155 and ERC-6909, ownership means the caller's `balanceOf(msg.sender, tokenId)` is positive.

2. **Delegated control through delegate.xyz v2, ERC-721 only.** For ERC-721 tokens, the caller is a delegate of the owner registered on the canonical delegate.xyz v2 registry at `0x00000000000000447e69651d841bD8D104Bed493`. The adapter MUST verify the delegation by calling `checkDelegateForERC721` for the specific `(owner, tokenContract, tokenId)`. Delegation is not recognized for ERC-1155 or ERC-6909 in this version of the ERC.

If `tokenContract == address(0)`, the call MUST revert. If the caller satisfies neither control rule, the call MUST revert.

### Signature-authorized registration

`counterfactualRegisterWithSig` separates the caller from the authorizer, so a registration can be folded into a transaction the token owner does not send, such as a mint function that registers the agent at mint time. The token `owner` signs the payload and any caller MAY submit it.

Authorization on this path is by signature, not by `msg.sender`. The adapter MUST verify the signature and MUST verify that `owner` is the current direct owner of the token (rule 1 in Authorization). Delegated control (rule 2) MUST NOT be accepted on the signed path.

The signed payload MUST be an [EIP-712](./eip-712.md) typed message. The domain is:

```solidity
EIP712Domain(string name, string version, uint256 chainId, address verifyingContract)
```

where `chainId` is `block.chainid` and `verifyingContract` is the adapter address. The message type is:

```solidity
CounterfactualRegister(
    uint8 standard,
    address tokenContract,
    uint256 tokenId,
    string agentURI,
    MetadataEntry[] metadata,
    address agentWallet,
    address owner,
    uint256 expiration
)
MetadataEntry(string metadataKey, bytes metadataValue)
```

The message is hashed per the EIP-712 rules for these types. Together, `chainId` and `verifyingContract` in the domain and `tokenContract` and `tokenId` in the message bind the signature to one chain, one adapter, and one token, so it cannot be replayed across chains, adapters, or tokens. The adapter MUST accept both EOA signatures and [ERC-1271](./eip-1271.md) contract signatures over the derived digest.

The signature is bounded by time rather than by a nonce. The adapter MUST reject the call if `expiration` is more than an implementation-defined maximum delay past the current block timestamp, and MUST reject the call if the current block timestamp is past `expiration`. This bounds the window in which a superseded-but-unexpired signature can be replayed; latest-event-wins (see Event consumption) resolves ordering.

`agentWallet` is optional. When it is non-zero, the adapter MUST emit `CounterfactualAgentWalletSet` for the same `registrationHash` in the same call, binding the wallet in the same transaction as the registration.

### Function behavior

`registrationHash(tokenContract, tokenId)` MUST return the hash defined in Registration hash.

`counterfactualRegister(standard, tokenContract, tokenId, agentURI, metadata)` MUST verify caller control. It MUST reject reserved metadata keys as described in Reserved metadata keys. It MUST compute `registrationHash`, emit `CounterfactualAgentRegistered`, and return the computed hash. The overload without `metadata` MUST behave as if an empty metadata array had been supplied.

`counterfactualRegisterWithSig(standard, tokenContract, tokenId, agentURI, metadata, agentWallet, owner, expiration, signature)` MUST verify the signature and `expiration` as described in Signature-authorized registration, and MUST verify that `owner` is the current direct owner of the token. It MUST reject reserved metadata keys as described in Reserved metadata keys. It MUST compute `registrationHash`, emit `CounterfactualAgentRegistered` with `emitter = owner`, emit `CounterfactualAgentWalletSet` with `emitter = owner` when `agentWallet != address(0)`, and return the computed hash.

An implementation MAY declare the metadata-bearing overload as `public` and MAY use `memory` rather than `calldata` for internal-call ergonomics, provided the external ABI is equivalent.

`counterfactualSetAgentURI(standard, tokenContract, tokenId, newURI)` MUST verify caller control. It MUST compute `registrationHash` and emit `CounterfactualAgentURISet`.

`counterfactualSetMetadata(standard, tokenContract, tokenId, metadataKey, metadataValue)` MUST verify caller control. It MUST reject reserved metadata keys as described in Reserved metadata keys. It MUST compute `registrationHash` and emit `CounterfactualMetadataSet`.

`counterfactualSetMetadataBatch(standard, tokenContract, tokenId, metadata)` MUST verify caller control. It MUST reject reserved metadata keys as described in Reserved metadata keys. It MUST compute `registrationHash` and emit `CounterfactualMetadataBatchSet`.

`counterfactualSetAgentWallet(standard, tokenContract, tokenId, newWallet)` MUST verify caller control. It MUST compute `registrationHash` and emit `CounterfactualAgentWalletSet`. This function MUST NOT require an ERC-8004 agent-wallet signature because no ERC-8004 registry wallet binding is created.

`counterfactualUnsetAgentWallet(standard, tokenContract, tokenId)` MUST verify caller control. It MUST compute `registrationHash` and emit `CounterfactualAgentWalletUnset`.

### Reserved metadata keys

The metadata key used by the adapter to represent the canonical external-token binding is reserved. A counterfactual registration or metadata write MUST revert if any supplied metadata entry attempts to write that key.

Implementations SHOULD use `agent-binding` as the reserved key when they also expose ERC-8004 adapter bindings. Implementations MAY reserve an equivalent binding key if their adapter binding metadata uses a different canonical name, but the reserved key MUST be documented by the implementation.

The metadata key `cf-registration` is also reserved. A counterfactual registration or metadata write MUST revert if any supplied metadata entry attempts to write that key. The key's semantics are defined in Promotion metadata key and apply only on the canonical ERC-8004 identity-registry record, never through counterfactual events.

### Promotion metadata key

When an adapter promotes a counterfactual identity to a canonical ERC-8004 registration by minting an ERC-8004 identity token for the same external token, the adapter MUST set the reserved metadata key `cf-registration` on the resulting ERC-8004 identity-registry record. The value MUST be the 32-byte `registrationHash` defined in Registration hash, encoded as `bytes` in the registry's metadata format.

`cf-registration` gives the canonical ERC-8004 identity record a deterministic, registry-rooted back-link to its counterfactual event history. Because `registrationHash` includes `block.chainid`, the emitting adapter's `address(this)`, and the declared token `standard`, the value uniquely identifies the originating counterfactual record across chains, adapter deployments, and declared standards without requiring any external lookup.

`cf-registration` is permissively writable by the current binding controller. Indexers validate the back-link per the rules in Security Considerations; the latest valid value supersedes earlier values.

## Rationale

This ERC defines a counterfactual agent identity as an event-anchored claim about an external token, distinct from an ERC-8004 identity-registry token. Event logs are onchain consensus output, so emission gives indexers a verifiable surface for discovery and profile state. Indexers and clients reconstruct the agent state by applying the latest event per `registrationHash`.

The `registrationHash` includes `block.chainid` and `address(this)` to prevent replay across chains and adapter deployments. It includes `tokenContract` and `tokenId` because those coordinates identify the external object that controls the counterfactual identity. It includes `standard` because a single token contract can serve more than one ERC token standard. Including `standard` in the hash separates counterfactual identities by declared standard for the same `(tokenContract, tokenId)` pair, and prevents a claim under one standard from colliding with a claim under another standard at the same coordinates.

Each event carries `uint8 version` as the first non-indexed field. This makes every event self-describing without consuming a topic slot. Indexers decode the three indexed identity fields, read the payload version at a fixed ABI position, and choose the correct decoder for the remaining payload.

The version field follows a forward-compatible bump scheme. Implementations conforming to this ERC emit `1`. Future revisions should bump the value only on payload-structure breakage, such as reordered or removed fields, and not on purely additive trailing fields.

An indexed version would allow topic filtering, but it would consume a scarce topic that is better reserved for identity coordinates. A single global version on `CounterfactualAgentRegistered` would be smaller, but it would force indexers to retain registration context before decoding later URI, metadata, or wallet events. That requirement conflicts with the stateless latest-event-wins model.

The counterfactual agent-wallet function does not require the ERC-8004 wallet signature used by the registry because it does not create a registry wallet binding. It is a counterfactual claim emitted onchain by the current external-token controller. It remains subject to replacement by later authorized events.

The `cf-registration` promotion key is written on the canonical ERC-8004 identity-registry record. Routing the back-link through registry metadata, rather than through an additional event, keeps the linkage alongside the canonical record itself, so it is visible to every consumer that reads the registry's metadata. The value is expected to be the same `registrationHash` already used as the indexer key for counterfactual events, which makes promotion a deterministic join across the two surfaces with no additional protocol state when indexer validation succeeds. Reserving the key against counterfactual writes prevents a counterfactual emitter from fabricating a promotion link before the canonical mint has occurred.

## Backwards Compatibility

This ERC does not change ERC-8004 registry contracts and does not change ERC-721 identity token behavior. It defines a separate adapter surface that emits onchain counterfactual events.

Existing deployments that emitted unversioned counterfactual events are not ABI-compatible with the event signatures in this ERC. Indexers that support both forms should decode legacy unversioned events separately. They should treat this ERC's `version == 1` events as the canonical standardized payload.

Counterfactual registrations do not allocate ERC-8004 registry storage. Applications that require a registry-allocated ERC-8004 identity token continue to use ERC-8004 registration flows.

## Test Cases

Reference test coverage for an implementation of this ERC is available in the Adapter8004 Foundry tests:

- `test/security/Adapter8004.counterfactual.t.sol`
- `test/Adapter8004.delegate.t.sol`

Implementations should include tests covering at least:

1. `registrationHash(tokenContract, tokenId)` returns `keccak256(abi.encode(block.chainid, adapter, tokenContract, tokenId))`, and is identical for the same token regardless of the `standard` supplied to the counterfactual functions.
2. Each successful counterfactual function emits exactly one version `1` event.
3. Calls revert when `tokenContract == address(0)`, the token standard is unsupported, or the caller does not control the external token.
4. Single and batch metadata writes revert for the reserved binding metadata key.
5. For ERC-721, a caller delegated through delegate.xyz v2 is accepted exactly when `checkDelegateForERC721` returns true for the `(owner, tokenContract, tokenId)`, and rejected otherwise. Delegation calls for ERC-1155 and ERC-6909 are rejected.

## Reference Implementation

A reference implementation of the unversioned counterfactual function family is available in the Adapter8004 implementation at `src/Adapter8004.sol`.

The event declarations are available in the Adapter8004 interface at `src/interfaces/IERC8004AdapterCounterfactual.sol`.

Implementations of this ERC add the `uint8 version` field specified above to each counterfactual event and emit `version == 1`.

The promotion flow is out of scope for this reference implementation; future revisions may add it.

## Security Considerations

A counterfactual event proves only that the emitting caller satisfied the adapter's control rule when the event was emitted, captured onchain as a log entry. A counterfactual registration is not an ERC-8004 identity-registry token and does not prove historical ownership. Indexers and applications MUST NOT treat a counterfactual claim as a canonical ERC-8004 registration or as proof that the caller still controls the external token.

The authorization check is the primary security boundary. Implementations MUST perform the check before emitting events. Implementations MUST fail closed for unsupported token standards. Implementations SHOULD apply conservative rules when recognizing delegated control.

Latest-event-wins semantics mean a current token controller can replace prior counterfactual URI, metadata, and wallet claims. Applications that cache counterfactual data SHOULD track block number, transaction index, and log index, and SHOULD handle chain reorganizations.

The reserved binding metadata key prevents a caller from fabricating the adapter's canonical binding metadata through counterfactual metadata events. Implementations MUST reject that key in counterfactual registration and in single and batch metadata writes.

The `cf-registration` key is reserved against counterfactual writes. Counterfactual events MUST NOT carry this key in their metadata payload. On the canonical side, the adapter permits the current binding controller to write any value to `cf-registration`. Indexers MUST validate each `cf-registration` back-link before treating it as a legitimate promotion. The validation requires ALL of the following:

1. The decoded value corresponds to a `registrationHash` whose counterfactual events exist on this chain via this adapter.
2. The latest counterfactual event emitter for that hash is consistent with the current owner of the canonical ERC-8004 record, the entity controlling the bound external token.
3. The `setMetadata` write that placed the current `cf-registration` value was authorized by the current binding controller at the time of the write.

If any condition fails, the indexer MUST ignore the back-link. The latest valid write supersedes earlier valid writes; consumers SHOULD track block number, transaction index, and log index for ordering.

Counterfactual wallet events are counterfactual claims, not registry wallet bindings. Applications SHOULD display them as counterfactual or unverified unless their own policy accepts token-control-gated wallet claims. A later ERC-8004 registry wallet binding, if created, is a separate registry-side authorization with its own signature requirements.

Event payload versions do not make old decoders safe by themselves. Indexers SHOULD reject unknown `version` values unless they support the corresponding payload schema.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

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
requires: 8004, 721
---

## Abstract

Counterfactual registration gives any tokenized object an instant onchain agent identity. The controller of any ERC-721, ERC-1155, or ERC-6909 token can emit a versioned event from a compliant adapter contract and immediately become discoverable as an agent, carrying an `agentURI`, agent wallet, and arbitrary metadata, without minting a registry token or paying storage gas. Counterfactual ERC-8004 registration is designed for external tokens that need discoverability and the ability to manage public-facing agent records without creating a new ownership token.

Every counterfactual identity is keyed by a deterministic `registrationHash` derived from the chain, the emitting adapter, and the external token coordinates. Indexers and clients resolve state by applying the latest event for a given `registrationHash`, making updates to URI, metadata, and wallet as cheap as a log emission and as immediate as the next block. A counterfactual identity can later be promoted to a canonical ERC-8004 registration when value at stake justifies the additional storage cost, but the counterfactual layer alone is enough to power discovery, profile rendering, agent-to-agent messaging, and reputation signal flow.

## Motivation

ERC-8004 gives agents an onchain identity-registry token, but some agents are already anchored by external NFTs, semi-fungible tokens, or other tokenized rights. Requiring those agents to mint a second identity-registry token before discovery adds cost, coordination, and lifecycle friction.

Counterfactual registration lets a token controller make an onchain ERC-8004 identity claim without minting an ERC-8004 identity token and without allocating registry storage. The claim is indexable, displayable, updatable, and can later be promoted into a full ERC-8004 registration through a separate adapter flow. Because this ERC's counterfactual path uses event emission instead of registry storage, it avoids registry storage writes, avoids calls into the ERC-8004 registry contract, and keeps authority tied to current control of the external token.

Versioned events are required so indexers can safely decode this ERC's payloads even if future revisions add or reorder fields.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

This ERC defines counterfactual registration via event emission. Future ERCs MAY define alternative counterfactual emission mechanisms, for example registration carried in transaction calldata, that preserve the same authorization, hashing, latest-write-wins, and no-registry-storage-write properties. Compliant indexers for those future ERCs would consume the alternate mechanism while maintaining the same `registrationHash` semantics.

### Interface

Compliant adapters MUST implement the following interface:

```solidity
pragma solidity ^0.8.24;

interface IERC8004CounterfactualRegistration {
    enum TokenStandard {
        ERC721,
        ERC1155,
        ERC6909
    }

    struct MetadataEntry {
        string metadataKey;
        bytes metadataValue;
    }

    /// @notice Computes the canonical counterfactual registration hash.
    function registrationHash(address tokenContract, uint256 tokenId) external view returns (bytes32);

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

The `TokenStandard` enum values are `ERC721 == 0`, `ERC1155 == 1`, and `ERC6909 == 2`. Implementations MUST support `ERC721`. Implementations MAY support additional token standards. If an implementation does not support a supplied `standard`, the call MUST revert.

Every counterfactual event defined in this ERC MUST emit `version == 1`. Implementations MAY expose the current payload version through an additional pure function or constant, but consumers MUST rely on the per-event `version` field when decoding logs.

### Registration hash

For every counterfactual function, the adapter MUST compute:

```solidity
registrationHash = keccak256(abi.encode(block.chainid, address(this), tokenContract, tokenId));
```

`address(this)` is the emitting adapter address. The hash binds a counterfactual identity to one chain, one adapter, and one external token. `standard` is intentionally not part of the hash; it is event payload context.

`registrationHash(address tokenContract, uint256 tokenId)` MUST be deterministic and MUST NOT depend on mutable contract storage. Apart from `block.chainid` and `address(this)`, it MUST use only its function arguments.

### Event emission behavior

Counterfactual functions MUST NOT write adapter storage. Counterfactual functions MUST NOT call the ERC-8004 identity registry. Counterfactual functions MUST NOT mint an ERC-721 identity token, MUST NOT allocate an `_owners[tokenId]` slot on the ERC-8004 registry, and MUST NOT set `_tokenURIs[tokenId]` on the ERC-8004 registry. The emitted event log entry is the onchain output of a counterfactual call defined by this ERC. Each successful counterfactual function call MUST emit exactly one event from this ERC's counterfactual event set.

Indexers and clients MUST treat counterfactual state as state derived from onchain logs. For a given `registrationHash`, the latest valid event of each event type supersedes earlier events of that type. For metadata, indexers and clients MUST treat the latest valid write for a given metadata key as authoritative. A `CounterfactualMetadataBatchSet` event contains independent writes for each entry in `metadata`.

Indexers and clients SHOULD order events by block number, transaction index, and log index. Indexers and clients SHOULD persist the `version` of the last-applied event for each derived field so replay under a different payload version can be flagged instead of silently overwriting decoded state. If a chain reorganization removes an event, indexers and clients MUST remove the derived state from that event and replay the surviving canonical log sequence.

### Authorization

For every counterfactual function, the caller MUST currently control the external token identified by `standard`, `tokenContract`, and `tokenId`. The adapter MUST perform this authorization check at call time, before emitting any counterfactual event.

For ERC-721 tokens, control means the caller is the current owner of `tokenId` or otherwise satisfies the adapter's ERC-721 binding-control rule. An adapter MAY recognize delegated control for ERC-721 tokens. For ERC-1155 and ERC-6909 tokens, if supported, control means the caller holds a positive balance of `tokenId`, unless the adapter defines a stricter control rule.

If `tokenContract == address(0)`, the call MUST revert. If the caller does not currently control the external token, the call MUST revert.

### Function behavior

`registrationHash(tokenContract, tokenId)` MUST return the hash defined in [Registration hash](#registration-hash).

`counterfactualRegister(standard, tokenContract, tokenId, agentURI, metadata)` MUST verify caller control, MUST reject reserved metadata keys as described in [Reserved metadata keys](#reserved-metadata-keys), MUST compute `registrationHash`, MUST emit `CounterfactualAgentRegistered`, and MUST return the computed hash. The overload without `metadata` MUST behave as if an empty metadata array had been supplied.

An implementation MAY declare the metadata-bearing overload as `public` and may use `memory` rather than `calldata` for internal-call ergonomics, provided the external ABI is equivalent.

`counterfactualSetAgentURI(standard, tokenContract, tokenId, newURI)` MUST verify caller control, MUST compute `registrationHash`, and MUST emit `CounterfactualAgentURISet`.

`counterfactualSetMetadata(standard, tokenContract, tokenId, metadataKey, metadataValue)` MUST verify caller control, MUST reject reserved metadata keys as described in [Reserved metadata keys](#reserved-metadata-keys), MUST compute `registrationHash`, and MUST emit `CounterfactualMetadataSet`.

`counterfactualSetMetadataBatch(standard, tokenContract, tokenId, metadata)` MUST verify caller control, MUST reject reserved metadata keys as described in [Reserved metadata keys](#reserved-metadata-keys), MUST compute `registrationHash`, and MUST emit `CounterfactualMetadataBatchSet`.

`counterfactualSetAgentWallet(standard, tokenContract, tokenId, newWallet)` MUST verify caller control, MUST compute `registrationHash`, and MUST emit `CounterfactualAgentWalletSet`. This function MUST NOT require an ERC-8004 agent-wallet signature because no ERC-8004 registry wallet binding is created.

`counterfactualUnsetAgentWallet(standard, tokenContract, tokenId)` MUST verify caller control, MUST compute `registrationHash`, and MUST emit `CounterfactualAgentWalletUnset`.

### Reserved metadata keys

The metadata key used by the adapter to represent the canonical external-token binding is reserved. A counterfactual registration or metadata write MUST revert if any supplied metadata entry attempts to write that key.

Implementations SHOULD use `agent-binding` as the reserved key when they also expose ERC-8004 adapter bindings. Implementations MAY reserve an equivalent binding key if their adapter binding metadata uses a different canonical name, but the reserved key MUST be documented by the implementation.

The metadata key `counterfactual-registration` is also reserved. A counterfactual registration or metadata write MUST revert if any supplied metadata entry attempts to write that key. The key's semantics are defined in [Promotion metadata key](#promotion-metadata-key) and are written only at promotion time on the canonical ERC-8004 identity-registry record, never through counterfactual events.

### Promotion metadata key

When an adapter promotes a counterfactual identity to a canonical ERC-8004 registration by minting an ERC-8004 identity token for the same external token, the adapter MUST set the reserved metadata key `counterfactual-registration` on the resulting ERC-8004 identity-registry record. The value MUST be the 32-byte `registrationHash` defined in [Registration hash](#registration-hash), encoded as `bytes` in the registry's metadata format.

`counterfactual-registration` gives the canonical ERC-8004 identity record a deterministic, registry-rooted back-link to its counterfactual event history. Because `registrationHash` includes `block.chainid` and the emitting adapter's `address(this)`, the value uniquely identifies the originating counterfactual record across chains and adapter deployments without requiring any external lookup.

`counterfactual-registration` SHOULD be treated as immutable once written. Implementations that allow metadata edits on canonical ERC-8004 records MUST NOT permit overwriting this key.

## Rationale

This ERC uses events rather than registry storage because counterfactual registrations are onchain claims about an external token, not ERC-8004 identity-registry tokens. Event logs are onchain consensus output, so this ERC's emission mechanism gives indexers a verifiable discovery and profile-state surface while avoiding ERC-8004 registry writes, ERC-721 identity token mints, `_owners[tokenId]` allocations, `_tokenURIs[tokenId]` writes, and adapter SSTOREs.

The `registrationHash` includes `block.chainid` and `address(this)` to prevent replay across chains and adapter deployments. It includes `tokenContract` and `tokenId` because those coordinates identify the external object that controls the counterfactual identity. `standard` is excluded so the same token coordinates have one counterfactual identity key; the declared standard remains available in registration events for authorization and indexing context.

Each event carries `uint8 version` as the first non-indexed field. This makes every event self-describing without spending a topic slot. Indexers can decode the three indexed identity fields consistently, read the payload version at a fixed ABI position, and then choose the correct decoder for the remaining payload. This is a forward-compatible bump scheme: implementations conforming to this ERC emit `1`, and future revisions should bump only on payload-structure breakage, such as reordered or removed fields, not on purely additive trailing fields. An indexed version would allow topic filtering but would consume a scarce topic that is better reserved for identity coordinates. A single global version on `CounterfactualAgentRegistered` would be smaller, but it would force indexers to retain registration context before decoding later URI, metadata, or wallet events, which conflicts with the stateless latest-event-wins model.

The counterfactual agent-wallet function does not require the ERC-8004 wallet signature used by the registry because it does not create a registry wallet binding. It is a counterfactual claim emitted onchain by the current external-token controller and remains subject to replacement by later authorized events.

The `counterfactual-registration` promotion key is written only at promotion time on the canonical ERC-8004 identity-registry record. Routing the back-link through registry metadata, rather than through an additional event, ensures the linkage lives alongside the canonical record itself and is visible to every consumer that reads the registry's metadata. The value is the same `registrationHash` already used as the indexer key for counterfactual events, so promotion is a deterministic join across the two surfaces with no additional protocol state. Reserving the key against counterfactual writes prevents a counterfactual emitter from fabricating a promotion link before the canonical mint has actually occurred.

## Backwards Compatibility

This ERC does not change ERC-8004 registry contracts and does not change ERC-721 identity token behavior. In particular, no `_owners[tokenId]` slot on the ERC-8004 registry is allocated and no `_tokenURIs[tokenId]` on the ERC-8004 registry is set by this ERC. It defines a separate adapter surface that emits additional onchain events.

Existing deployments that emitted unversioned counterfactual events are not ABI-compatible with the event signatures in this ERC. Indexers that support both forms should decode legacy unversioned events separately and should treat this ERC's `version == 1` events as the canonical standardized payload.

Because counterfactual registrations do not allocate ERC-8004 registry storage, applications that require a registry-allocated ERC-8004 identity token need to use ERC-8004 registration flows.

## Test Cases

Reference test coverage for an implementation of this ERC is available in the Adapter8004 Foundry tests:

- `test/security/Adapter8004.counterfactual.t.sol`
- `test/Adapter8004.delegate.t.sol`

Implementations should include tests covering at least:

1. `registrationHash(tokenContract, tokenId)` returns `keccak256(abi.encode(block.chainid, adapter, tokenContract, tokenId))`.
2. Each successful counterfactual function emits exactly one version `1` event and does not write adapter or ERC-8004 registry storage.
3. Calls revert when `tokenContract == address(0)`, the token standard is unsupported, or the caller does not control the external token.
4. Single and batch metadata writes revert for the reserved binding metadata key.
5. Delegate-based ERC-721 control, when implemented, is accepted only under the adapter's documented binding-control rule.

## Reference Implementation

A reference implementation of the unversioned counterfactual function family is available in the Adapter8004 implementation at `src/Adapter8004.sol`.

The event declarations are available in the Adapter8004 interface at `src/interfaces/IERC8004AdapterCounterfactual.sol`.

Implementations of this ERC add the `uint8 version` field specified above to each counterfactual event and emit `version == 1`.

## Security Considerations

Counterfactual registrations are not ERC-8004 identity-registry tokens; no `_owners[tokenId]` is set on the registry and no `_tokenURIs[tokenId]` is written on the registry. They also do not prove historical ownership. They prove only that the emitting caller satisfied the adapter's control rule at the time the event was emitted, captured onchain as a log entry. Indexers and applications must not treat a counterfactual claim as a registry write or as proof that the caller still controls the external token.

The authorization check is the primary security boundary. Implementations need to perform the check before emitting events, fail closed for unsupported token standards, and be conservative when recognizing delegated control.

Latest-event-wins semantics mean a current token controller can replace prior counterfactual URI, metadata, and wallet claims. Applications that cache counterfactual data should track block number, transaction index, and log index, and should handle chain reorganizations.

The reserved binding metadata key prevents a caller from fabricating the adapter's canonical binding metadata through counterfactual metadata events. Implementations need to reject that key in both single and batch metadata writes and during counterfactual registration.

The reserved `counterfactual-registration` promotion key prevents a counterfactual emitter from fabricating a promotion back-link before a canonical ERC-8004 mint has actually occurred. Counterfactual functions MUST reject this key the same way they reject the binding key. On the canonical side, implementations that allow metadata edits on ERC-8004 identity-registry records MUST refuse to overwrite an existing `counterfactual-registration` value. Consumers that rely on the back-link should verify that the value matches the deterministic `registrationHash` derived from the canonical record's bound external token and the writing adapter address.

Counterfactual wallet events are counterfactual claims, not registry wallet bindings. Applications should display them as counterfactual or unverified unless their own policy accepts token-control-gated wallet claims. A later ERC-8004 registry wallet binding, if created, is a separate registry-side authorization with its own signature requirements.

Event payload versions do not make old decoders safe by themselves. Indexers should reject unknown `version` values unless they explicitly support the corresponding payload schema.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

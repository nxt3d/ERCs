---
eip: 8217
title: Agent NFT Identity Bindings
description: A standard metadata record for binding ERC-8004 agents to external NFT controllers.
author: Prem Makeig (@nxt3d)
discussions-to: https://ethereum-magicians.org/t/add-erc-8217-agent-nft-identity-bindings/28339
status: Draft
type: Standards Track
category: ERC
created: 2026-04-05
requires: 8004
---

## Abstract

This ERC defines a standard onchain metadata record and verification interface for expressing that an [ERC-8004](./erc-8004.md) agent identity is bound to an external NFT or tokenized asset contract. The metadata record stores the binding contract, token standard, token contract, and token identifier in a compact binary format under a reserved metadata key. The binding contract and token contract MAY be different contracts or the same contract.

## Motivation

[ERC-8004](./erc-8004.md) defines an identity registry for agents, but it does not define a standard way to express that control of an agent is delegated to or mediated by another token contract. In practice, adapters and collection contracts may hold the ERC-8004 registration while allowing ownership or balances of an external token to determine who may manage the agent record.

Without a standard metadata format:

- clients cannot reliably discover that an agent is controlled through an external binding contract
- marketplaces and wallets cannot decode bound-token information consistently
- indexers must support adapter-specific formats

This ERC provides a canonical metadata key, binary encoding, and verification interface so clients can discover the binding contract, decode the bound token, and verify the canonical bound-token record for an agent, whether the binding logic lives in a separate adapter contract or directly in the token contract itself.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Simplified Interface

Binding adapters compliant with this ERC MUST expose a minimal interface that allows clients to:

- retrieve the stored binding for an agent id

The required interface is:

```solidity
pragma solidity ^0.8.24;

interface IERCAgentBindings {
    enum TokenStandard {
        ERC721,
        ERC1155,
        ERC6909
    }

    struct Binding {
        TokenStandard standard;
        address tokenContract;
        uint256 tokenId;
    }

    function bindingOf(uint256 agentId) external view returns (Binding memory);
}
```

Contracts MAY expose richer functions such as registration, URI updates, metadata updates, wallet binding, or administrative upgrade controls, but these are outside the verification scope of this ERC.

### Metadata Key

Implementations compliant with this ERC MUST store the binding record under the [ERC-8004](./erc-8004.md) metadata key:

```text
agent-binding
```

### Binding Record Format

The metadata value for `agent-binding` MUST be encoded as:

```solidity
abi.encodePacked(bindingContract, tokenStandard, tokenContract, tokenIdLength, compactTokenId)
```

The binary layout is:

```text
┌─────────────────┬───────────────┬───────────────┬───────────────┬────────────────┐
│ bindingContract │ tokenStandard │ tokenContract │ tokenIdLength │ compactTokenId │
└─────────────────┴───────────────┴───────────────┴───────────────┴────────────────┘
```

The fields have the following meanings:

- `bindingContract`: 20-byte EVM address of the binding or adapter contract that governs the relationship; this MAY equal `tokenContract`
- `tokenStandard`: 1-byte enum describing the bound token standard
- `tokenContract`: 20-byte EVM address of the bound token contract
- `tokenIdLength`: 1-byte unsigned integer describing the number of bytes in `compactTokenId`
- `compactTokenId`: variable-length minimal big-endian encoding of the bound token id

### Token Standard Enum

The `tokenStandard` byte MUST use the following values:

- `0x00`: ERC-721
- `0x01`: ERC-1155
- `0x02`: ERC-6909

Values outside this set are reserved for future extensions and MUST NOT be emitted by implementations compliant with this version of the ERC.

### Compact Token ID Encoding

The `compactTokenId` field MUST encode the token id in minimal big-endian form:

- if `tokenId == 0`, `tokenIdLength` MUST be `0` and `compactTokenId` MUST be omitted
- if `tokenId > 0`, `tokenIdLength` MUST equal the minimum number of bytes required to represent the token id
- leading zero bytes MUST NOT be included

Examples:

- token id `0` encodes as `tokenIdLength = 0`
- token id `5` encodes as `tokenIdLength = 1`, `compactTokenId = 0x05`
- token id `0x1234` encodes as `tokenIdLength = 2`, `compactTokenId = 0x1234`

### Required Behavior

An implementation that uses this ERC to represent a binding for an [ERC-8004](./erc-8004.md) agent:

1. MUST write the binding record under the `agent-binding` key
2. MUST ensure the binary payload matches the format defined above
3. MUST treat `agent-binding` as a reserved key and prevent untrusted callers from overwriting it arbitrarily
4. MAY define any control semantics it wants in the binding contract itself, including ERC-721 ownership or ERC-1155 / ERC-6909 balance-based control

This ERC standardizes discovery and canonical binding verification only. It does not standardize authorization rules inside the binding contract.

### Verification Flow

Clients verifying an [ERC-8004](./erc-8004.md) binding under this ERC MUST:

1. read the `agent-binding` metadata from the [ERC-8004](./erc-8004.md) registry
2. decode `bindingContract`, `tokenStandard`, `tokenContract`, and `tokenId`
3. call `bindingOf(agentId)` on `bindingContract`
4. verify that the returned binding matches the decoded metadata

If any step fails, clients MUST treat the binding relationship as unverified.

The `bindingContract` and `tokenContract` MAY be different addresses or the same address. Clients MUST NOT assume they are distinct.

### Example Encoding

For:

- `bindingContract = 0x1111111111111111111111111111111111111111`
- `tokenStandard = 0x00`
- `tokenContract = 0x2222222222222222222222222222222222222222`
- `tokenId = 0x1234`

the metadata payload is:

```text
0x
1111111111111111111111111111111111111111
00
2222222222222222222222222222222222222222
02
1234
```

## Rationale

### Why store the binding contract?

The token contract and token id alone are not sufficient. The same token may be interpreted differently by different adapter or binding contracts. Including the binding contract makes the control system explicitly discoverable and lets clients inspect or query the contract that actually defines the authorization rules.

In some implementations, the token contract itself defines the binding logic. In those cases, `bindingContract` and `tokenContract` are the same address.

### Why include the token standard as an enum byte?

The token contract address and token id do not reveal whether control should be interpreted through `ownerOf`, `balanceOf`, or another standard-specific rule. A one-byte enum is compact and matches how adapter implementations typically branch between ERC-721, ERC-1155, and ERC-6909 behavior.

### Why use compact token ids?

Token ids are often small. Encoding them in 32 bytes wastes storage and calldata. A length-prefixed compact integer preserves unambiguous decoding while significantly reducing size for common cases.

## Backwards Compatibility

This ERC is backwards compatible with:

- [ERC-8004](./erc-8004.md), because it only standardizes one metadata key and value format
- [ERC-721](./erc-721.md), [ERC-1155](./erc-1155.md), and ERC-6909 binding schemes, because it does not alter their token semantics

Existing [ERC-8004](./erc-8004.md) registries and adapters are not required to support this metadata key, but implementations that do can interoperate on a common discovery format.

## Test Cases

Expected encodings:

ERC-721 example with token id `0`:

```text
bindingContract = 0x1111111111111111111111111111111111111111
tokenStandard   = 0x00  // ERC-721
tokenContract   = 0x2222222222222222222222222222222222222222
tokenId         = 0

=> 0x111111111111111111111111111111111111111100222222222222222222222222222222222222222200
```

ERC-1155 example with token id `5`:

```text
bindingContract = 0x1111111111111111111111111111111111111111
tokenStandard   = 0x01  // ERC-1155
tokenContract   = 0x2222222222222222222222222222222222222222
tokenId         = 5

=> 0x11111111111111111111111111111111111111110122222222222222222222222222222222222222220105
```

ERC-721 example with token id `0x1234`:

```text
bindingContract = 0x1111111111111111111111111111111111111111
tokenStandard   = 0x00  // ERC-721
tokenContract   = 0x2222222222222222222222222222222222222222
tokenId         = 0x1234

=> 0x1111111111111111111111111111111111111111002222222222222222222222222222222222222222021234
```

## Security Considerations

Clients MUST NOT assume that decoding `agent-binding` alone is sufficient to determine the current controller of an agent. The metadata reveals the binding contract and the bound token, but control semantics remain implementation-specific and may depend on current ownership, balances, thresholds, delegated permissions, or additional contract logic.

Clients MUST:

1. decode the binding metadata
2. inspect or query the referenced `bindingContract`
3. verify the canonical binding with `bindingOf(agentId)`

Implementations MUST reserve the `agent-binding` metadata key so that untrusted callers cannot overwrite or forge the canonical record after registration.

This metadata format is EVM-address based. It does not describe non-EVM bindings and does not itself encode chain context.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

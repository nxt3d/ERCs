---
eip: XXXX
title: Tag Registry for Ethereum
description: A general-purpose singleton tag registry for Ethereum contracts and entities
author: Prem Makeig (@nxt3d)
discussions-to: https://ethereum-magicians.org/t/erc-xxxx-tag-registry-for-ethereum/25722
status: Draft
type: Standards Track
category: ERC
created: 2025-11-07
requires: 8049
---

## Abstract

This standard defines a singleton Tag Registry on Ethereum mainnet for storing semantic labels that can be referenced by [ERC-8004](./erc-8004.md) AI agents, ENS names, smart contracts, NFTs, and other on-chain entities. Tag definitions are stored globally in the singleton registry using [ERC-8049](./erc-8049.md) contract-level metadata. On-chain entities reference up to 255 tags using a compact uint16 encoding that fits up to 15 tags efficiently in a single storage slot. This enables wallets, indexers, and applications to discover and categorize agents, ENS names, and other on-chain entities without relying on external APIs or custom metadata schemes.

## Motivation

The Ethereum ecosystem lacks a universal vocabulary for categorizing and discovering on-chain entities. [ERC-8004](./erc-8004.md) AI agents, ENS names, smart contracts, NFTs, and other entities each use custom metadata schemes for classification, fragmenting the ecosystem and preventing cross-platform discovery. For example:

- AI agents in [ERC-8004](./erc-8004.md) registries have no standard way to categorize themselves onchain (e.g., "chatbot", "defi", "nft")
- ENS names lack standardized categorization for services they represent
- Smart contracts have no universal tags for their purpose or domain

Without a shared tagging vocabulary, each application must invent its own classification scheme, leading to:

- Duplicate effort across projects
- Incompatible categorization systems
- Difficulty discovering related entities (e.g., all DeFi agents or gaming ENS names) across applications
- Reliance on centralized indexing services

A standardized tagging system provides:

1. **Universal vocabulary**: A single canonical tag list that all applications can reference
2. **Efficient storage**: Compact uint16 encoding keeps on-chain tag references affordable
3. **Extensibility**: Optional descriptions support rich metadata without breaking compatibility
4. **Interoperability**: Tags work across any agent, ENS name, contract, NFT, or entity

This standard defines a singleton Tag Registry deployed on Ethereum mainnet. The registry uses [ERC-8049](./erc-8049.md) contract-level metadata to store tag definitions. Onchain entities can reference these tags using compact onchain metadata, enabling universal discovery and categorization.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.html) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174.html).

### Tag Registry Singleton

This standard defines a singleton Tag Registry contract deployed on Ethereum Mainnet. The registry serves as the canonical source for all tag definitions that can be referenced by [ERC-8004](./erc-8004.md) AI agents, ENS names, smart contracts, NFTs, and other on-chain entities.

The registry contract:
- MUST implement [ERC-8049](./erc-8049.md) for contract-level metadata storage
- SHOULD be deployed at a deterministic address for easy discovery
- MUST be immutable once deployed, though tag definitions can be added or updated by the contract owner

### Tag Definitions

Tag definitions are stored in the Tag Registry using contract-level metadata from [ERC-8049](./erc-8049.md). 

Each tag is identified by a unique Tag ID (an unsigned integer in the range `[1, 65535]`) which is encoded in the metadata key. The stored values are:

- **Tag Label**: A UTF-8 string (e.g., `"defi"`, `"nft"`, `"gaming"`, `"dao"`)
- **Tag Description** *(optional)*: Human-readable text explaining the tag

#### Metadata Key Format

For a given `tagId`, convert it to a zero-padded 5-digit decimal string:
- `1` becomes `"00001"`
- `47` becomes `"00047"`  
- `947` becomes `"00947"`
- `65535` becomes `"65535"`

Store the tag label using the metadata key:

```
id-tag-<id>
```

The value MUST be the label encoded as UTF-8 bytes.

**Example**: Tag ID `47` with label `"defi"` is stored at key `"id-tag-00047"` with value `0x64656669` (UTF-8 bytes for "defi").

#### Optional Description

Store an optional description using the key:

```
id-tag-description-<id>
```

The value MUST be UTF-8 bytes containing the description text.

**Example**: Tag ID `47` with description `"Decentralized finance applications and protocols"` is stored at key `"id-tag-description-00047"` with value `0x446563656e7472616c697a65642066696e616e6365206170706c69636174696f6e7320616e642070726f746f636f6c73` (UTF-8 bytes for the description).

### Tag References

Onchain entities such as [ERC-8004](./erc-8004.md) agents, ENS names, contracts, NFTs, etc. can reference tags from the registry by storing tag IDs using a compact encoding format. The encoding supports up to 255 tag IDs, with 15 or fewer tags fitting efficiently within a single storage slot for optimal gas efficiency.

Entities MAY store tag references in their own metadata systems:
- **[ERC-8004](./erc-8004.md) agents**: Use Onchain Metadata with key `"agent-id-tags"`
- **ENS names**: Can store tags in ENS `data()` records (ENSIP-24). The exact key SHOULD be defined in an ENSIP standard. 
- **Smart contracts**: Can implement custom storage or use [ERC-8049](./erc-8049.md) contract metadata. For contracts using ERC-8049 the key SHOULD be `"contract-id-tags"`. 
- **NFTs**: Can use token-level Onchain Metadata [ERC-8048](./erc-8048.md) and SHOULD use key `"nft-id-tags"` for each token. 

#### Encoding Format

Given an array of distinct tag IDs, encode them as:

```
bytes = [count] [tag_1] [tag_2] ...
```

Where:
- `count` is a single byte containing the number of tags (0-255)
- Each tag is exactly 2 bytes: the `uint16` tag ID (0-65535) encoded as bytes.

**Empty tag list**: Either a single zero byte `0x00` or empty bytes `""` (zero length)

#### Encoding Example

Assigning tags `[47, 118, 2049]` using `abi.encodePacked`:

```solidity
// In Solidity
bytes memory result = abi.encodePacked(
    uint8(3),      // count
    uint16(47),    // 0x002f
    uint16(118),   // 0x0076
    uint16(2049)   // 0x0801
);
// Result: 0x03002f00760801
```

#### Decoding

To decode a tag list:

1. If the payload is empty (zero length), interpret as an empty tag list
2. Otherwise, read the first byte as `count`
3. For each tag (0 to count-1):
   - Read 2 bytes as a `uint16` tag ID

Parsers MUST reject payloads where:
- Payload length is not 0 and not exactly `1 + (count * 2)` bytes
- `count` is zero but additional bytes follow

### Tag Registry Interface

The singleton Tag Registry contract on Ethereum mainnet MUST provide the following interface for managing tags. The registry implements [ERC-8049](./erc-8049.md) for storing tag definitions at the contract level. 

#### Interface

```solidity
pragma solidity ^0.8.25;

interface IERCXXXXTagRegistry is IERC8049ContractMetadata {
    /// @notice Emitted when a tag is defined or updated
    /// @param tagId The tag identifier
    /// @param label The UTF-8 tag label
    /// @param description Optional UTF-8 description (may be empty)
    event TagDefined(uint256 indexed tagId, string label, string description);

    /// @notice Define or update a tag
    /// @param tagId The tag identifier (1-65535)
    /// @param label UTF-8 tag label
    /// @param description Optional UTF-8 description
    function defineTag(uint256 tagId, bytes calldata label, bytes calldata description) external;

    /// @notice Look up a tag definition
    /// @param tagId The tag identifier
    /// @return label The UTF-8 tag label
    /// @return description The UTF-8 description (may be empty)
    function getTag(uint256 tagId) external view returns (bytes memory label, bytes memory description);
}
```

### Required Behavior

The registry contract MUST enforce these validation rules:

1. **Tag ID range**: Tag IDs MUST be in the range [1, 65535]
2. **Non-empty labels**: Tag labels MUST NOT be empty
3. **Access control**: Setting and updating tag labels and definitions MUST be restricted to the contract owner

The registry maps each tag ID to a single definition. Defining a tag with an existing ID overwrites the previous definition. The contract owner is responsible for managing tag definitions and avoiding unintentional overwrites.

Tag labels MAY use any capitalization, for example “DeFi”, and SHOULD avoid leading or trailing whitespace for consistency.

When encoding tag references:

1. **No duplicate tags**: Tag ID arrays MUST NOT contain duplicate tag IDs
2. **Count limit**: Encoded payloads MUST NOT contain more than 255 tags (the maximum value of a uint8 count byte)
3. **Valid tag IDs**: Each tag ID MUST be in the range [1, 65535]

## Rationale

### Singleton Registry Design

Deploying a single canonical Tag Registry on Ethereum mainnet ensures all applications reference the same tag definitions, preventing fragmentation. A singleton provides:
- **Universal vocabulary**: All on-chain entities use the same tag IDs
- **Network effects**: As more tags are defined, the registry becomes more valuable to all participants
- **Simplified discovery**: Applications only need to know one contract address
- **Immutable trust anchor**: The singleton can be deployed with transparent ownership visible to all

### Fixed Key Prefixes

Using deterministic keys (`id-tag-00047`) allows indexers to enumerate all tags by scanning for the `id-tag-*` prefix without needing to rely on indexers. This makes tags discoverable even when historical event logs are unavailable.

### Compact Encoding

The fixed 2-byte format provides a simple, deterministic encoding while supporting tag IDs up to 65535. For example:
- A single tag costs 3 bytes (1 byte count + 2 bytes tag ID)
- 15 tags cost exactly 31 bytes (1 + 15*2)
- 255 tags cost exactly 511 bytes (1 + 255*2)

This makes frequent tag updates affordable while keeping parsing straightforward.

### Tag Ordering

Tag IDs can be stored in any order within the encoded payload. Implementations SHOULD ignore duplicate tag IDs. 

### Separate Description Storage

Storing descriptions in separate metadata keys keeps the core tag-to-label mapping lean. Applications that only need to display labels can skip description lookups entirely.

## Backwards Compatibility

This standard is fully compatible with:

- **NFTs and Token Registries**: NFTs that implement [ERC-8048](./erc-8048.md) Onchain Metadata may support tags using Onchain Metadata.  
- **[ERC-8004](./erc-8004.md) Trustless Agents**: Can use tags to categorize trustless agents. 
- **ENS**: Resolvers that support `data()` records can add tags to ENS profiles. 

## Reference Implementation

This reference implementation demonstrates the singleton Tag Registry contract deployed on Ethereum mainnet. The contract implements ERC-8049 for storing tag definitions.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import {ERC8049ContractMetadata} from "./ERC8049ContractMetadata.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {Strings} from "@openzeppelin/contracts/utils/Strings.sol";

/**
 * @title TagRegistry
 * @notice Singleton Tag Registry for Ethereum contracts and entities
 * @dev Extends ERC8049ContractMetadata for contract-level metadata storage of tag definitions
 */
contract TagRegistry is ERC8049ContractMetadata, IERCXXXXTagRegistry, Ownable {
    using Strings for uint256;
    // Tag-specific event
    event TagDefined(uint256 indexed tagId, string label, string description);

    constructor(address initialOwner) Ownable(initialOwner) {}

    /**
     * @notice Set contract metadata (restricted to owner)
     * @param key The metadata key
     * @param value The metadata value as bytes
     */
    function setContractMetadata(string calldata key, bytes calldata value) external onlyOwner {
        _setContractMetadata(key, value);
    }

    /**
     * @notice Define or update a tag
     * @param tagId Unique tag identifier (1 to 65535)
     * @param label Lowercase UTF-8 label (e.g., "chatbot")
     * @param description Optional UTF-8 description
     */
    function defineTag(
        uint256 tagId,
        string calldata label,
        string calldata description
    ) external onlyOwner {
        require(tagId > 0 && tagId <= 65535, "Tag ID must be 1-65535");
        require(bytes(label).length > 0, "Label cannot be empty");

        // Store label using _setContractMetadata
        string memory key = string.concat("id-tag-", _zeroPad(tagId));
        _setContractMetadata(key, bytes(label));

        // Store optional description
        if (bytes(description).length > 0) {
            string memory descKey = string.concat("id-tag-description-", _zeroPad(tagId));
            _setContractMetadata(descKey, bytes(description));
        }

        emit TagDefined(tagId, label, description);
    }

    /**
     * @notice Get a tag definition
     * @param tagId The tag identifier
     * @return label The UTF-8 tag label as a string
     * @return description The UTF-8 description as a string (may be empty)
     */
    function getTag(uint256 tagId) external view returns (string memory label, string memory description) {
        string memory key = string.concat("id-tag-", _zeroPad(tagId));
        label = string(this.getContractMetadata(key));

        string memory descKey = string.concat("id-tag-description-", _zeroPad(tagId));
        description = string(this.getContractMetadata(descKey));
    }

    /**
     * @dev Convert tag ID to zero-padded 5-digit decimal string
     * @param tagId The tag identifier (0-65535)
     * @return 5-character decimal string (e.g., "00047" for tag 47)
     */
    function _zeroPad(uint256 tagId) internal pure returns (string memory) {
        string memory str = tagId.toString();
        bytes memory strBytes = bytes(str);
        
        if (strBytes.length >= 5) {
            return str;
        }
        
        bytes memory result = new bytes(5);
        uint256 offset = 5 - strBytes.length;
        
        // Pad with '0'
        for (uint256 i = 0; i < offset; i++) {
            result[i] = 0x30;
        }
        
        // Copy digits
        for (uint256 i = 0; i < strBytes.length; i++) {
            result[offset + i] = strBytes[i];
        }
        
        return string(result);
    }
}
```

The singleton Tag Registry contract is deployed once on Ethereum mainnet at a canonical address. Onchain entities ([ERC-8004](./erc-8004.md) agents, ENS names, contracts, NFTs, etc.) can reference tags by storing tag IDs using the compact encoding format in their own metadata systems. For example, [ERC-8004](./erc-8004.md) agents can use the `"agent-id-tags"` metadata field, while ENS names can use a `data()` record specified using an ENSIP (not specified here). The registry provides read-only access to tag definitions for all applications.

## Security Considerations

None.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

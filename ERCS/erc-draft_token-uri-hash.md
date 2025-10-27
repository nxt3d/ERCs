---
eip: XXXX
title: Token URI Hash for Onchain Metadata Verification
description: A standard field for storing a truncated hash of token URI JSON data to enable verification of offchain URL queried JSON data.
author: Prem Makeig (@nxt3d)
discussions-to: 
status: Draft
type: Standards Track
category: ERC
created: 2025-10-27
requires: 8048
---

## Abstract

This ERC defines a standard metadata field for storing hashes of token URI JSON data. It enables smart contracts and clients to verify that mutable off-chain metadata fetched from tokenURIs has not been altered, providing data integrity verification without requiring full onchain storage.

## Motivation

ERC-8004, ERC-721, ERC-1155, ERC-6909 and other token standards allow agents to reference off-chain metadata via tokenURI. When these URIs point to mutable data sources, there is no way to verify that the fetched JSON data matches what was originally specified. This creates security risks when for example ERC-8004 agents rely on mutable metadata for critical operations.

This ERC provides a lightweight solution by storing a hashes of the JSON data onchain as metadata, enabling verification that off-chain data matches the onchain commitment without requiring the full data to be stored onchain.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Scope

This ERC extends ERC-8048's Onchain Metadata for token registries that use ERC-8048, including ERC-8004 trustless agents, ERC-721 (ERC721 Metadata JSON Schema), ERC-1155 (ERC-1155 Metadata URI JSON Schema), and ERC-6909 (Asset Metadata), by defining a specific metadata field for token URI hash verification. 

### Metadata Field Specification

#### Field Name

Contracts implementing this ERC MUST support a metadata field with the key:

`"token-uri-hash"`

#### Field Value Format

The value MUST be a bytes value with the following structure:

- **Byte 1**: Hash type identifier (i.e. `0x00` for Keccak256)
- **Bytes 2-N**: Truncated hash value (length depends on hash algorithm)

#### Hash Types

##### Keccak256

Currently, only Keccak256 (hash type `0x00`) is defined. For Keccak256, the first 30 bytes of the hash are stored, and the last 2 bytes are discarded. Future hash types MAY be specified in this ERC or in a future registry.

| Hash Type | Name | Description |
|-----------|------|-------------|
| `0x00` | Keccak256 | First 30 bytes of Keccak256 hash (32 bytes truncated to 30) |
| `0x01` - `0xFF` | Reserved | Reserved for future hash algorithms |

### Hash Computation

When computing the hash for storage in the `token-uri-hash` field:

1. Fetch the JSON data from the tokenURI as UTF-8 encoded text
2. Locate the first `{` character and the last `}` character in the fetched data
3. Extract exactly the bytes from the first `{` through and including the last `}`, including all whitespace, newlines, and spaces between the braces (excluding any data before the first `{` or after the last `}`)
4. Compute the hash of these exact UTF-8 bytes using the specified hash algorithm
5. Truncate if necessary (for Keccak256, take first 30 bytes)
6. Prepend the hash type byte (e.g., 0x00 for Keccak256) to create the final value

### Verification Process

To verify that off-chain JSON data matches the onchain commitment:

**Step 1**: Query the tokenURI and retrieve the JSON data as UTF-8 encoded text

**Step 2**: Read the `token-uri-hash` metadata field (up to 31 bytes)

**Step 3**: Extract the hash type from byte 1 of the metadata value (e.g., `0x00` for Keccak256)

**Step 4**: Follow the hash computation process (steps 1-6) to compute the token-uri-hash value

**Step 5**: Compare the computed token-uri-hash value with the stored `token-uri-hash` value byte-by-byte

**Step 6**: If the values match exactly, the JSON data is verified as unaltered

### Example: Keccak256

Consider an ERC-8004 agent with the following tokenURI JSON (agent registration file):

```json
{
  "type": "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  "name": "DeFi Trading Agent",
  "description": "An autonomous trading agent for DeFi operations",
  "image": "https://example.com/agentimage.png",
  "endpoints": [
    {
      "name": "A2A",
      "endpoint": "https://agent.example/.well-known/agent-card.json",
      "version": "0.3.0"
    }
  ],
  "registrations": [
    {
      "agentId": 12345,
      "agentRegistry": "eip155:1:0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb7"
    }
  ],
  "supportedTrust": ["reputation", "crypto-economic"]
}
```

The verification process:

1. **Hash Computation**:
   - Extract exactly the bytes from the first `{` through and including the last `}`: The entire JSON object including all fields like `type`, `name`, `description`, `image`, `endpoints`, `registrations`, and `supportedTrust`, preserving all whitespace and newlines as they appear between these delimiters
   - Compute Keccak256 hash of the UTF-8 bytes: `0xb2b415319466d217991e86f39cff067cdada1318e913a2e8460e2fc9b5a204b4`
   - Take first 30 bytes: `0xb2b415319466d217991e86f39cff067cdada1318e913a2e8460e2fc9b5a`
   - Prepend hash type byte (0x00 for Keccak256): `0x00b2b415319466d217991e86f39cff067cdada1318e913a2e8460e2fc9b5a`

2. **Verification**: After computing the hash, check it against the stored value using `getMetadata(12345, "token-uri-hash")`. If the result matches `0x00b2b415319466d217991e86f39cff067cdada1318e913a2e8460e2fc9b5a`, the data is verified.

## Rationale

Standards that rely on off-chain JSON data referenced by `tokenURI` (such as ERC-8004) often use traditional HTTP endpoints where clients cannot verify that fetched data hasn't been altered by compromised servers or DNS attacks. While IPFS offers content-addressing with built-in integrity verification, many implementations prefer or require traditional web hosting for flexibility, mutability, or infrastructure simplicity. For such HTTP-based deployments, HTTPS only secures transport but doesn't prove content integrity. By storing a hash of the canonical JSON data onchain, this standard enables trustless verification of off-chain data regardless of hosting method.

## Security Considerations

None.

## Backwards Compatibility

Fully compatible with ERC-8004, and compatible with token registries that support tokenURI and ERC-8048 onchain metadata.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).


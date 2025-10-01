---
title: Backup URIs (Optional Extension for ERC-721, ERC-6909, ERC-8004)
description: A compact, URI-based specification for providing backup URIs as fallback metadata sources.
author: nxt3d (@nxt3d)
discussions-to: <URL>
status: Draft
type: Standards Track
category: ERC
created: 2025-09-30
---

## Abstract

This ERC defines the `backup-uris` metadata key for use by ERC-721, ERC-6909, and ERC-8004 compliant contracts that implement the getMetadata function (ERC-XXXX). The metadata key provides a standardized way to specify backup URIs as fallback metadata sources. The value is a list of backup URIs separated by semicolons, allowing clients to fallback to alternative metadata sources when the primary source is unavailable.

## Motivation

This ERC addresses the need for reliable metadata access by providing backup URI fallbacks while maintaining compatibility with existing ERC-721, ERC-6909, and ERC-8004 standards. By providing multiple URI sources, we ensure metadata availability even when primary sources fail. The backup URIs are additional fallback sources that clients can try when the main tokenURI fails, not a replacement for the primary URI. Rather than embedding complex fallback logic in contracts, this approach provides a standardized way to specify backup sources, allowing clients to implement robust fallback mechanisms.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Scope

This ERC is an optional extension that MAY be implemented by any ERC-721, ERC-6909, or ERC-8004 compliant contract that exposes the required metadata function. It is particularly useful for ensuring metadata availability and reliability.

### Required Metadata Function

Contracts implementing this ERC MUST expose the following function:

```solidity
interface IOnchainMetadata {
    /// @notice Get metadata value for a key (bytes of UTF-8 unless specified otherwise).
    function getMetadata(uint256 tokenId, bytes calldata key) external view returns (bytes memory);
}
```

- `getMetadata(tokenId, bytes("backup-uris"))`: Returns the backup URIs for the given token ID as UTF-8 encoded bytes

### Data Format

- **Encoding**: All backup URI data MUST be UTF-8 encoded bytes.
- **Delimiter**: Semicolon (`;`) separates individual URIs.
- **URI validation**: Each URI MUST conform to RFC 3986.
- **Order**: URIs are tried in the order they appear in the list.

**Example**:
```
https://example.com/metadata/1;https://backup.example.com/metadata/1;ipfs://QmHash/metadata.json
```

### Client Behavior

1. Call `getMetadata(tokenId, bytes("backup-uris"))` to retrieve backup URIs.
2. Split the returned UTF-8 string by semicolons (`;`) to get individual URIs.
3. Try each URI in order until one succeeds.
4. If all URIs fail, handle according to client's error policy.

## Rationale

This design prioritizes reliability, simplicity, and compatibility with existing standards. The following design decisions support these goals:

- Semicolon delimiter avoids conflicts with URI characters while being visually clear.
- UTF-8 encoding ensures international compatibility and standard text handling.
- Ordered list provides predictable fallback behavior.
- Simple scheme name clearly indicates the purpose.

## Backwards Compatibility

- Fully compatible with ERC-721, ERC-6909, and ERC-8004.
- Clients that do not support backup URIs will simply not use them, falling back to their default error handling when primary URIs fail.

## Reference Implementation

The interface is defined in the Required Metadata Function section above. Implementations should follow the standard ERC-721, ERC-6909, or ERC-8004 patterns while adding the required metadata function.

## Security Considerations

Clients MUST validate each URI before attempting to fetch metadata to prevent malicious redirects or invalid requests. URIs should be checked for proper scheme validation and domain allowlists as appropriate for the client's security requirements.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

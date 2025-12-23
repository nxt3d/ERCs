---
eip: XXXX
title: Decentralized Chain Identifier (DCID)
description: A self-derived, authority-free identifier scheme for uniquely identifying blockchain networks.
author: Prem Makeig (@nxt3d)
discussions-to: 
status: Draft
type: Standards Track
category: ERC
created: 2025-12-23
---

## Abstract

This ERC defines a Decentralized Chain Identifier (DCID) scheme for uniquely identifying blockchain networks across ecosystems. A DCID is a self-derived, opaque identifier computed from a human-meaningful label and an explicit salt, hashed and truncated to 16 bytes, and encoded as lowercase hexadecimal. The scheme is authority-free, collision-tolerant, and designed to scale to a future where chains may be created dynamically and ephemerally. It is intentionally simple, relying only on two input components: a label and a salt.

## Motivation

Existing chain identification schemes rely on centrally assigned numeric identifiers or informal registries. These approaches do not scale well to environments with many Layer 2 networks, application-specific chains, testnets, forks, and ephemeral deployments.

A decentralized chain identifier should avoid central issuance or allocation, be stable, deterministic, and easy to compute, remain compact and tooling-friendly, avoid numeric interpretation and ordering, and allow collision resolution without governance intervention. This ERC draws lessons from IP addressing, DNS, and Ethereum address design, favoring oversized identifier space, opaque semantics, and deterministic derivation.

DCIDs build on the work in [ERC-7785](./erc-7785.md) on hash-derived chain identifiers, but intentionally keep derivation simpler, use a fixed 16-byte identifier for more manageable handling, and leave registries, domains, and labeling schemes out of scope.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Overview

A DCID is derived from the following inputs:

- A label, constrained to DNS hostname label syntax
- A salt, chosen by the chain operator
- A fixed hash function
- Truncation to 16 bytes
- Lowercase hexadecimal encoding

DCIDs are self-issued. No registry or approval process is required.

### Label Syntax

The DCID label MUST follow the DNS hostname label rules defined in RFC 1035 and RFC 1123, with the following constraints:

- Allowed characters: `a`–`z`, `0`–`9`, `-`
- Lowercase only
- MUST NOT begin or end with a hyphen
- MUST contain at least one character
- MUST NOT contain dots or any hierarchy

Formally, the label MUST match the following regular expression:

```
^[a-z0-9](?:[a-z0-9-]*[a-z0-9])?$
```

Labels are opaque identifiers. They do not imply ownership, hierarchy, or governance. To assist chains in selecting labels, an informational registry MAY be used; however, issuers are ultimately responsible for choosing a `(label, salt)` pair that derives a unique DCID.

Examples of valid labels:

- `ethereum`
- `optimism`
- `base`
- `examplechain`
- `examplechain-testnet`

### Salt

The salt is an explicit, non-negative integer chosen by the chain operator.

The salt exists to support issuing a very large number of DCIDs per label, as well as:

- Chain restarts
- Forks
- Project failure and succession
- Collision resolution

No global coordination of salt values is required. Reuse of labels with different salts is expected.

### Derivation

A DCID is derived as follows:

1. Encode the label as UTF-8 bytes.
2. Encode the salt as a big-endian 256-bit unsigned integer (32 bytes).
3. Concatenate `label || salt`.
4. Compute the hash using `keccak256`.
5. Truncate the result to the first 16 bytes.
6. When representing the DCID as text, encode the result as lowercase hexadecimal; the `0x` prefix SHOULD be included.

In pseudocode:

```
dcid = "0x" + hex_lower(keccak256(label || salt)[0:16])
```

### Examples

The following examples use the derivation specified above (salt encoded as a 32-byte big-endian unsigned integer):

| Label | Salt (uint256) | DCID (first 16 bytes of keccak256) |
| :--- | ---: | :--- |
| `ethereum` | 1 | `0x0a1e714977db7edb6b42c34fe96dcfe0` |
| `solana` | 0 | `0x5c27dede7502c8a2617b1fc85b0c20e7` |
| `optimism` | 0 | `0x8fe27659879241e47d93735068b5ccca` |

### Encoding and Representation

- DCIDs MUST be represented as 16 bytes (128 bits).
- When represented as text, DCIDs SHOULD be encoded as lowercase hexadecimal with a `0x` prefix (34 characters total).
- DCIDs MUST be treated as opaque identifiers.
- Equality comparison is the only defined operation.
- DCIDs MUST NOT be interpreted as numeric values.

### Collision Handling

Collisions are extremely unlikely due to the 128-bit identifier space. In the event of a collision, a new salt MUST be chosen and a new DCID derived.

Collision resolution does not require protocol changes, registries, or coordination beyond the affected parties.

### Registries

Optional registries MAY exist to map human-readable names or metadata to DCIDs. Such registries are informational only and MUST NOT be considered authoritative for DCID validity.

The DCID itself is the sole identifier.

## Rationale

As the number of chains grows (including L2s, app-specific chains, forks, and ephemeral deployments), centralized allocation and informal lists do not scale. DCID provides a simple, self-issued identifier derived from only a label and a salt, with a compact fixed 16-byte value that is easy for software to store, compare, and log without relying on registries, domains, or naming systems.

## Backwards Compatibility

This ERC introduces a new identifier scheme and does not modify existing Ethereum chain ID semantics. DCIDs can coexist with numeric chain IDs; applications MAY support both identification schemes.

## Security Considerations

DCIDs are not intended to provide cryptographic security guarantees; issuers are responsible for avoiding collisions by choosing a unique `(label, salt)` pair. Informational registries MAY be used to coordinate and manage labels, but ultimate collision handling is achieved by selecting a unique salt value for a given label, yielding a new DCID.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).


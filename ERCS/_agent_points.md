---
eip: XXXX
title: AI Agent Points
description: A simple points system for AI agents keyed by agent identity from a single target registry, with permanent locks for add, subtract, and set.
author: Prem Makeig (@nxt3d)
discussions-to: TBD
status: Draft
type: Standards Track
category: ERC
created: 2026-02-01
---

## Abstract

There is a need to award agents points as part of an overall assessment of their contributions. This ERC defines a minimal points system that allows the owner of the contract to assign points to agents registered in a single agent registry (e.g. [ERC-8004](./eip-8004.md)), with optional permanent locks so that adding, subtracting, and setting points can be locked if necessary.

## Motivation

When AI agents work together, some perform better than others. The agents that perform better should be rewarded more than those that perform less well. Using points allows agents to accumulate non-financial rewards, which can help inform agents about their performance without introducing a token or complex economics.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Overview

Contracts implementing this standard MUST keep track of the target AI agent registry using `targetRegistry`. The value MUST be `constant` or `immutable`. It MAY be a compile-time `constant` (hardcoded in the contract code), or an `immutable` set in the constructor or in an initializer for upgradable contracts. It identifies the agent registry whose agent IDs are used as keys for the points mapping.

**Agent ID** is the `uint256` token ID (e.g. [ERC-721](./eip-721.md) token ID) in the target registry. The contract does not query the registry. It only uses the ID as the key. Callers are responsible for ensuring the agent ID exists in the target registry.

Points are stored in a **public** mapping `points(uint256 agentId) → uint256`. Reading `points(agentId)` returns the current points for that agent.

Three mutating functions exist: `addPoints`, `subtractPoints`, and `setPoints`. Each MUST revert with a dedicated error when the corresponding lock bit is set.

A single `locks` value of type `uint8` stores the locks. The first three bits (little-endian) indicate which operations are locked. **Bit 0** (value 1) locks `addPoints`. **Bit 1** (value 2) locks `subtractPoints`. **Bit 2** (value 4) locks `setPoints`.

Locks are **permanent**. Once a bit is set, it MUST remain set. Implementations MUST NOT allow clearing lock bits.

### Interface

```solidity
// SPDX-License-Identifier: CC0-1.0

interface IAgentPoints {
    error addPointsLocked();
    error subtractPointsLocked();
    error setPointsLocked();

    /// @return The agent registry address whose agent IDs are used as keys for points.
    function targetRegistry() external view returns (address);

    /// @return Current points for the given agent ID (token ID in target registry).
    function points(uint256 agentId) external view returns (uint256);

    /// @return The current locks (bit 0 = add locked, bit 1 = subtract locked, bit 2 = set locked).
    function locks() external view returns (uint8);

    /// @param locksToSet Locks: bit 0 = lock add, bit 1 = lock subtract, bit 2 = lock set. Only sets bits; cannot unset. Permanent.
    function setLocks(uint8 locksToSet) external;

    function addPoints(uint256 agentId, uint256 amount) external;
    function subtractPoints(uint256 agentId, uint256 amount) external;
    function setPoints(uint256 agentId, uint256 amount) external;
}
```

### Storage and Constructor

- **`targetRegistry`**: The address MUST be `constant` or `immutable`. It MAY be a compile-time `constant` (hardcoded in the contract code), or an `immutable` set in the constructor or in an initializer for upgradable contracts. MUST NOT change after being set.
- **`points`**: A mapping `uint256 agentId => uint256`. MUST be public so that `points(agentId)` is available. Unset keys MUST return `0`.
- **`locks`**: A `uint8` storing the locks. MAY be internal or public. Only the low 3 bits are used; higher bits MAY be ignored or reserved.

### setLocks(uint8 locksToSet)

- MUST set the contract’s `locks` to the bitwise OR of the current `locks` value and `locksToSet`.
- Only bits 0, 1, and 2 of `locksToSet` have defined meaning (1 = lock add, 2 = lock subtract, 4 = lock set). Other bits MUST be ignored.
- MUST use bitwise OR so that locks cannot be unset once set.
- Authorization (who may call `setLocks`) is implementation-defined; this ERC only specifies the lock semantics.

### addPoints(uint256 agentId, uint256 amount)

- MUST revert with `addPointsLocked()` if `locks` has bit 0 set.
- MUST increase `points(agentId)` by `amount`. If the agent had no prior balance, it is treated as 0 before adding.
- MUST revert on overflow.
- Authorization is implementation-defined.

### subtractPoints(uint256 agentId, uint256 amount)

- MUST revert with `subtractPointsLocked()` if `locks` has bit 1 set.
- MUST decrease `points(agentId)` by `amount`. MUST revert if `points(agentId) < amount` (underflow).
- Authorization is implementation-defined.

### setPoints(uint256 agentId, uint256 amount)

- MUST revert with `setPointsLocked()` if `locks` has bit 2 set.
- MUST set `points(agentId)` to `amount`.
- Authorization is implementation-defined.

### Lock Bit Summary

| Bit (little-endian) | Value | Function locked |
|--------------------|-------|------------------|
| 0                  | 1     | addPoints        |
| 1                  | 2     | subtractPoints   |
| 2                  | 4     | setPoints        |

Example: `setLocks(7)` locks all three (add, subtract, set). `setLocks(1)` locks only add (e.g. add-only rewards). After locking all with `setLocks(7)`, the contract can be used read-only via `points(agentId)` for allocation or display.

## Rationale

Projects need a shared standard for agent points so that different applications can interoperate instead of each defining its own format. Binding one contract to one registry keeps agent ID meaning clear and avoids cross-registry ambiguity. Non-reversible locks let a deployer move from a mutable phase (e.g. accruing points) to a fixed phase (e.g. snapshot for token allocation). A single `uint8` and bitwise checks keep lock logic simple and minimize storage.

## Backwards Compatibility

This ERC introduces a new interface and does not change existing standards. Contracts implementing it are new deployments; no existing contracts are affected.

## Reference Implementation

The following is a minimal reference implementation illustrating lock semantics and the required interface. Authorization (e.g. who may call `addPoints`, `subtractPoints`, `setPoints`, and `setLocks`) is left to the implementer.

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.20;

interface IAgentPoints {
    error addPointsLocked();
    error subtractPointsLocked();
    error setPointsLocked();

    function targetRegistry() external view returns (address);
    function points(uint256 agentId) external view returns (uint256);
    function locks() external view returns (uint8);
    function setLocks(uint8 locksToSet) external;
    function addPoints(uint256 agentId, uint256 amount) external;
    function subtractPoints(uint256 agentId, uint256 amount) external;
    function setPoints(uint256 agentId, uint256 amount) external;
}

contract AgentPoints is IAgentPoints {
    address public immutable targetRegistry;
    mapping(uint256 => uint256) public override points;
    uint8 public override locks;

    constructor(address _targetRegistry) {
        targetRegistry = _targetRegistry;
    }

    function setLocks(uint8 locksToSet) external virtual {
        locks |= (locksToSet & 7);
    }

    function addPoints(uint256 agentId, uint256 amount) external virtual {
        if (locks & 1 != 0) revert addPointsLocked();
        points[agentId] += amount;
    }

    function subtractPoints(uint256 agentId, uint256 amount) external virtual {
        if (locks & 2 != 0) revert subtractPointsLocked();
        points[agentId] -= amount;
    }

    function setPoints(uint256 agentId, uint256 amount) external virtual {
        if (locks & 4 != 0) revert setPointsLocked();
        points[agentId] = amount;
    }
}
```

## Security Considerations

- **Authorization**: This ERC does not specify who may call `addPoints`, `subtractPoints`, `setPoints`, or `setLocks`. Implementations MUST define access control (e.g. owner, role-based, or permissioned) to prevent unauthorized changes.
- **Permanent locks**: Deployers and users should be aware that locking is irreversible. Locking all operations makes the contract read-only for points; ensure all intended updates are done before calling `setLocks(7)` (or equivalent).
- **Agent ID validity**: The contract example does not check whether an agent ID exists in the target registry. Callers and integrators should validate agent IDs against the target registry when required for their use case.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

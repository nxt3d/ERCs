---
eip: TBD
title: Agent Delegations
description: A standard for establishing delegated relationships between accounts and AI agents using ERC-8092 Associated Accounts.
author: Prem Makeig (@nxt3d)
discussions-to: https://ethereum-magicians.org/t/erc-agent-delegations/XXXXX
status: Draft
type: Standards Track
category: ERC
created: 2025-12-15
requires: 8092
---

## Abstract

This ERC defines a standard for establishing delegated relationships between accounts and AI agents using [ERC-8092](./eip-8092.md) Associated Accounts. It specifies a JSON data format stored in the `data` field of the Associated Account Record that describes the agent's identity, capabilities, and endpoints. This enables AI agents to operate autonomously on the internet while maintaining a verifiable link to their owner/delegator.

## Motivation

As AI agents increasingly operate autonomously on the internet, users and services need to verify who owns an agent and what delegations it has. [ERC-8092](./eip-8092.md) provides a general framework for establishing relationships between two accounts, but does not specify the context for AI agent delegation.

This ERC extends ERC-8092 by:

1. Defining the "Delegated Agent" association type where the `initiator` is the delegator (owner) and the `approver` is the delegatee (agent)
2. Specifying a JSON data format for agent metadata that mirrors [ERC-8004](./eip-8004.md) (Trustless Agents)
3. Enabling agents to declare an ENS name for discoverable identity
4. Providing a standard way for services to verify agent ownership and capabilities

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Association Type

This ERC defines the "Delegated Agent" association type for ERC-8092 Associated Account Records:

- **`initiator`**: The delegator (owner) account that controls the agent
- **`approver`**: The delegatee (agent) account that operates on behalf of the owner

### Interface ID

The `interfaceId` field in the Associated Account Record MUST be set to the interface ID for `IDelegatedAgent`:

```solidity
interface IDelegatedAgent {
    function delegateAgent(
        string calldata association,
        string calldata name,
        string calldata description,
        string calldata image,
        string[] calldata endpoints
    ) external;
}
```

```solidity
bytes4 constant DELEGATED_AGENT_INTERFACE_ID = 0xa9ce26a1;
```

### Data Format

The `data` field in the Associated Account Record MUST contain UTF-8 encoded JSON stored as bytes. The JSON object MUST have the following structure:

```json
{
  "association": "Delegated Agent",
  "name": "myAgentName",
  "description": "A natural language description of the agent",
  "image": "https://example.com/agentimage.png",
  "endpoints": [
    {
      "name": "A2A",
      "endpoint": "https://agent.example/.well-known/agent-card.json"
    },
    {
      "name": "MCP",
      "endpoint": "https://mcp.agent.example/"
    },
    {
      "name": "ENS",
      "endpoint": "myagent.eth"
    }
  ]
}
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `association` | string | MUST be `"Delegated Agent"` for this ERC |
| `name` | string | The name of the agent (empty string is acceptable) |
| `description` | string | A natural language description of the agent (empty string is acceptable) |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `image` | string | URI to an image representing the agent |
| `endpoints` | array | Array of endpoint objects describing how to interact with the agent |

### Endpoints

The `endpoints` array MAY contain objects with the following structure:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | The protocol or service name (e.g., "A2A", "MCP", "ENS") |
| `endpoint` | string | The endpoint URI or identifier |
| `capabilities` | object | Optional capabilities object (for MCP endpoints) |

### ENS Identity

Agents SHOULD declare an ENS name in the `endpoints` array to enable identity discovery. The ENS profile can host additional data such as:

- Agent capabilities and skills
- Reputation records
- Verification credentials
- Extended metadata

Services resolving agent identity SHOULD check the ENS endpoint for additional information. If the ENS resolver implements ENSIP-24's `supportedDataKeys(bytes32 node)` function, clients can discover which data keys are available for the agent's profile.


### Encoding

To store the JSON data in the ERC-8092 `data` field:

```solidity
string memory jsonData = '{"association":"Delegated Agent","name":"MyAgent","description":"My AI agent",...}';
bytes memory data = bytes(jsonData);
```

### Validation

When consuming an Agent Delegation record, clients MUST:

1. Validate the ERC-8092 Associated Account Record according to that standard's rules
2. Verify the `interfaceId` is `0xa9ce26a1`
3. Parse the `data` field as UTF-8 JSON
4. Verify the `association` field is `"Delegated Agent"`
5. Verify required fields (`name`, `description`) are present (empty strings are acceptable)

## Rationale

This ERC builds on ERC-8092's robust framework for account associations rather than creating a new delegation mechanism. UTF-8 JSON provides human-readable, flexible data storage that mirrors ERC-8004's structure for consistency across the AI agent ecosystem. ENS integration enables decentralized, discoverable agent identities with rich metadata. Using ERC-8092's `initiator` as delegator and `approver` as delegatee aligns with the natural flow where the owner initiates delegation and the agent approves.

## Backwards Compatibility

No issues.

## Security Considerations

None.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).


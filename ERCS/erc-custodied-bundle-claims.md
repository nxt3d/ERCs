---
title: Custodied NFT Bundle Claims
description: Identify the NFT and inventory backing a bundle NFT and track delivery after the bundle NFT is burned
author: Prem Makeig (@nxt3d)
status: Draft
type: Standards Track
category: ERC
created: 2026-11-24
requires: 20, 165, 721, 1155, 6551
---

## Abstract

This standard bundles an NFT and the assets in its [ERC-6551](./erc-6551.md) account into a bundle NFT that can be bought and sold atomically. Without it, the seller can withdraw assets from the account before the sale settles, and an approved spender can withdraw them after it. The escrow takes custody of the NFT and the assets before the bundle NFT is listed. On redemption, the buyer receives the NFT and the assets move to a fresh account bound to it, where prior approvals do not apply. The seller can reclaim an unsold bundle after its deadline.

## Motivation

An agent NFT represents the identity of an AI agent. The NFT controls an ERC-6551 account that can hold fungible tokens or digital collectible NFTs. A buyer who values both the identity and the assets can verify the balance and still receive an empty account. The seller can withdraw the assets before the sale settles, and a spender with an existing approval can withdraw them even after the NFT changes hands, a risk shared by any NFT whose account holds value.

The solution is for the owner to take the agreed assets out of the ERC-6551 account, bundle them with the NFT in a permissionless escrow, and sell the resulting bundle NFT.

Eligible NFTs and the assets in their ERC-6551 accounts can be deposited without approval from an operator. On redemption, the assets move to a fresh account bound to the original NFT, where approvals from the old account do not apply.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

An **escrow** contract holds an NFT and its controlled assets together as a **bundle** for a fixed period. The controlled assets can come from an ERC-6551 account. A **bundle NFT** is minted for each bundle. It represents the claim to redeem that bundle. Each bundle receives a **bundle ID**. Its bundle NFT uses that number as its token ID. Bundle ID and token ID can therefore be used interchangeably. The **bundle NFT collection** is the separate ERC-721 contract that mints bundle NFTs.

### Discovery

The bundle NFT collection MUST implement [ERC-721](./erc-721.md). The [ERC-165](./erc-165.md) identifier of `ICustodiedBundleClaim` is `0x68420221`. The bundle NFT collection MUST return `true` from `supportsInterface(0x68420221)`. Each bundle NFT's token ID MUST equal its bundle ID. If the bundle ID exists, `bundleClaim(bundleId)` MUST return that bundle's record. After the bundle NFT is burned, the function MUST still return the record. If the bundle ID does not exist, it MUST revert.

```solidity
interface ICustodiedBundleClaim {
    function bundleClaim(uint256 bundleId) external view returns (
        address escrow, address vault, address controllingNFT,
        uint256 controllingTokenId, bytes32 manifestHash, uint8 state,
        uint256 heldMask, address seller, uint64 custodyDeadline
    );
}
```

Each deployment on a chain MUST use exactly one escrow. The `escrow` field returned by `bundleClaim(bundleId)` MUST be that escrow's address. The escrow MAY restrict which collections or assets it accepts. The escrow MUST accept deposits that meet those restrictions without requiring individual approval from a protocol operator.

### Inventory

The manifest lists the assets in a bundle. The escrow records each bundle and its manifest. The escrow manages listing, sale, redemption and reclaim. The vault is a separate contract that holds the deposited NFT and assets. Only its escrow can move them into or out of the vault. Every compliant escrow MUST implement `IBundleEscrow`. The escrow's `receipt()` MUST return the address of the bundle NFT collection.

```solidity
struct Entry {
    uint8 kind;
    address token;
    uint256 tokenId;
    uint256 amount;
}

interface IBundleEscrow {
    function receipt() external view returns (address);
    function getManifest(uint256 bundleId) external view returns (Entry[] memory);
    function redeemToFreshAccount(uint256 bundleId, bytes32 holderSalt) external returns (address account);
    function reclaim(uint256 bundleId, address extrasTo) external;
    function withdrawRemaining(uint256 bundleId, uint256 mask, bytes calldata data) external;
}
```

The escrow MUST take custody of the controlling NFT and every manifest entry before it makes the bundle NFT available. The assets can come from the NFT's ERC-6551 account or directly from the seller. The source ERC-6551 account is not an escrowed asset. Redemption does not deliver assets back to it. After deposit, the manifest MUST remain readable. It MUST also remain immutable.

Each manifest entry uses one of the following kinds.

| `kind` | Standard | `tokenId` | `amount` |
|---|---|---|---|
| 0 | [ERC-20](./erc-20.md) | MUST be 0 | MUST be greater than 0 |
| 1 | ERC-721 | The token's ID | MUST be 1 |
| 2 | [ERC-1155](./erc-1155.md) | The token's ID | MUST be greater than 0 |

A manifest MUST satisfy the following rules.

- It MUST hold at most 32 entries.
- It MUST NOT include the controlling NFT.
- It MUST NOT assign two kinds to the same token contract.
- Compare entries first by `kind` as a `uint8`, then by `token` as a `uint160`, then by `tokenId` as a `uint256`. The entries MUST be in strictly ascending order under this comparison. Strict ordering rejects duplicate entries. It also makes the same assets always produce the same manifest hash.

`MANIFEST_FORMAT_VERSION` is a `uint256` constant with value `2`. It identifies this manifest encoding. The variable `entries` holds the manifest as an `Entry[]` array. The manifest hash MUST be computed as follows.

```solidity
keccak256(abi.encode(MANIFEST_FORMAT_VERSION, block.chainid, escrow, controllingNFT, controllingTokenId, entries))
```

The hash records what was deposited. It stays the same as assets are delivered. Two deposits of the same assets produce the same hash. The bundle ID identifies the individual bundle.

### Bundle lifecycle

| State | Value | Action or entitlement |
|---|---:|---|
| None | 0 | No bundle |
| Deposited | 1 | The escrow holds the bundle NFT for the seller |
| Listed | 2 | The bundle NFT is available for an authorized primary sale |
| Issued | 3 | The sale transferred the bundle NFT to its recipient |
| Redeemed | 4 | Buyer exit complete |
| Reclaimed | 5 | Seller exit complete |
| Redeeming | 6 | Buyer exit started and something remains in custody |
| Reclaiming | 7 | Seller exit started and something remains in custody |

A sale MUST transfer the escrow-held bundle NFT to the recipient once. Subsequent holder transfers MUST revert. Only the current owner of the issued bundle NFT MAY initiate redemption. The escrow MUST burn the bundle NFT on exit. It MUST prevent a second exit. An approved operator MUST NOT be able to redeem on the owner's behalf.

The bundle NFT holder supplies `holderSalt`, a 32-byte contribution to the account salt. On redemption, the escrow MUST use this contribution to derive the account salt. The escrow MUST create a fresh [ERC-6551](./erc-6551.md) account with that salt. It MUST initialize the account. The account MUST be bound to the controlling NFT on this chain. The controlling NFT is due to the redeemer. The manifest entries are due to the new account. Before releasing entries to that account, the vault MUST establish that the NFT reached the redeemer. Before release, the vault MUST also establish that the redeemer controls the account. The recovery exception below permits the redeemer to select another asset destination if delivery of the NFT is pending.

At or after the custody deadline, only the recorded seller MAY reclaim a never-issued Deposited or Listed bundle. The controlling NFT MUST return to the seller. The seller selects a nonzero destination for the entries. Protocol custody contracts MUST be rejected as destinations. The seller may select the original account. Reclaiming entries can proceed while the controlling NFT is pending.

### Partial release and events

A successful exit transaction MAY leave assets in custody. Each set bit in `heldMask` MUST identify an outstanding delivery. Bits 0 through 31 correspond to manifest indices. Bit 32 corresponds to the controlling NFT. Other bits MUST be zero. A completed exit MUST have a zero mask. Burning the bundle NFT alone MUST NOT mark delivery complete.

Anyone MAY call `withdrawRemaining` with empty data to retry outstanding deliveries to the recorded destinations. The mask MUST reject bits outside the manifest and controlling NFT. Already released assets MUST NOT be delivered twice. A redirect uses nonempty data encoded as `abi.encode(uint8(1), destination)`. Only the recorded NFT recipient MAY redirect outstanding manifest entries to a valid destination. During redemption this is permitted only while the controlling NFT is still held. A redirect then permits delivery of entries independently of the NFT. On reclaim the seller can redirect while entries remain. The controlling NFT destination MUST remain fixed.

The vault MUST emit the following events for the corresponding actions. `ReleaseBegun` records the initial destinations. Its mode is 0 for redemption and 1 for reclaim. `EntryPending` reports an attempted transfer left outstanding in a successful transaction. A gated entry may remain held without an attempted transfer. `BundleEmptied` signals the last delivery. An indexer SHOULD reconcile events against the held mask at the same block.

```solidity
event ReleaseBegun(uint256 indexed bundleId, uint8 mode, address extrasTo, address nftTo);
event EntryReleased(uint256 indexed bundleId, uint256 indexed index, address to);
event EntryPending(uint256 indexed bundleId, uint256 indexed index);
event ControllingNFTReleased(uint256 indexed bundleId, address to);
event EntriesRedirected(uint256 indexed bundleId, address to);
event BundleEmptied(uint256 indexed bundleId);
```

### Marketplace display

Bundle NFT metadata SHOULD expose the following fields under `properties`. The key for the chain ID is `chain_id`. The key for the bundle ID is `bundle_id`. The keys for the original NFT contract and token ID are `original_nft_contract` and `original_nft_token_id`, respectively. The key for the manifest hash is `manifest_hash`. IDs SHOULD be decimal strings. The hash SHOULD be a 0x-prefixed 32-byte hex string. Before showing the assets, a marketplace SHOULD read the manifest at the escrow. It SHOULD then verify the manifest against the manifest hash. It SHOULD distinguish the original inventory from assets still held. It SHOULD label an issued bundle NFT as redeem only.

## Rationale

Marketplaces can display the bundle NFT using ERC-721. Calling `bundleClaim` on the bundle NFT collection supplies the escrow address. The returned record provides the information needed to verify the manifest. It also provides the information needed to explain a pending exit. A chain-scoped hash prevents accidental reuse across escrows. Fixed manifest indices let indexers follow each delivery separately. A fresh account separates the received assets from spending permissions granted by the old account. Marketplace order formats, fee schedules and royalty routing can evolve independently.

## Backwards Compatibility

Existing ERC-721 clients can display the bundle NFT. A client that assumes unrestricted secondary transfer will encounter a revert after issuance. The NewTrade bundle NFT collection on Base predates the discovery interface. Its metadata identifies the schema newtrade.bundle.v2. For that deployment, read `controller()` on the bundle NFT collection to find the escrow. Check that `receipt()` on the escrow returns the bundle NFT collection. Read `getBundle(id)` and `getManifest(id)` on the escrow. Read `bundleStatus(id)` on its vault for the release record and held mask. These reads remain useful after `tokenURI` and `ownerOf` cease to resolve a burned bundle NFT. The escrow's `receiptMetadata` callback is restricted to the bundle NFT collection.

## Test Cases

For chain 8453, escrow address(1), controlling collection address(2), token ID 7, and one entry (0, address(3), 0, 100), the manifest hash is 0x8794c8eeb370de4efc9912f10e0782dea78bfb9ae7c0c8c36105a11acfe50a1f. The initial held mask is 0x100000001. If the NFT reaches the redeemer and the asset remains held, state is 6 and the mask is 0x1. A successful retry for bit zero gives state 4 and a zero mask. The original manifest and hash remain available throughout.

## Reference Implementation

NewTrade on Base (chain ID 8453) illustrates the custody and release semantics. Contracts source revision cd990894e8fb331d0c42e4cbb853a59a24af01f6 documents this deployment. The deployed bundle NFT collection predates `bundleClaim` and its ERC-165 identifier.

| Contract | Base proxy |
|---|---|
| Escrow | 0xCc29F5c82C8b7c01Cac6036047E7413Df458f86d |
| Bundle NFT collection | 0x59900b207ea703B0bBcF8fD84C5E11dE46826185 |
| Custody vault | 0xe3a95e47B8ECC9F8E42a7BDE58dde2FBCbc69C53 |

The seller supplies the controlling NFT in both Base deposit paths. In `deposit`, the vault pulls every entry directly from the seller using token approvals. In `depositFromAccount`, all entries come from one executable account reporting the matching chain, collection and ID. This path requires enabled migration and an unused seller nonce. Each call selects one source for the entire manifest. A single deposit cannot combine assets held by the seller with assets held in the ERC-6551 account. Both paths lead to the same fresh-account redemption. The escrow snapshots the controlling NFT's creator royalty on listing and the bundle NFT collection quotes zero royalties. The escrow emits `BundleRedeemed` or `BundleReclaimed` at exit initiation even when entries remain held.

## Security Considerations

A collection can advertise any interface identifier. It can also serve arbitrary metadata. Clients need an independent basis for trusting a deployment and the behavior of its admitted tokens. The commitment covers selected entries and quantities. Unlisted balances, future transfers and asset value fall outside it. A token can refuse delivery indefinitely. Wallets should show the pending destination and held mask. They should continue to show both after the bundle NFT is burned. Redirecting assets changes where recovery delivers them. The beneficiary should explicitly confirm the redirect.

The Base escrow is a UUPS proxy. Administrators control fee and admission policy. Upgrade authorities can replace code governing existing custody. An interface check gives no assurance about those authorities. A controlling-collection allowlist can narrow admission to reviewed tokens. Each deployment has its own policy and trust assumptions.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

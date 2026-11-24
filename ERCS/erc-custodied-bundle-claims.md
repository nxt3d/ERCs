---
title: Custodied NFT Bundle Claims
description: Identify a claim on an NFT and its committed inventory and track delivery after the claim burns
author: Prem Makeig (@nxt3d)
status: Draft
type: Standards Track
category: ERC
created: 2026-11-24
requires: 20, 165, 721, 1155, 6551
---

## Abstract

When an NFT controls an [ERC-6551](./erc-6551.md) account, the seller can front run the sale by withdrawing assets just before it settles. ERC-20 approvals can also let the seller withdraw tokens after the sale.

Bundling places the controlling NFT and its assets in escrow. A bundle NFT, sold in place of the controlling NFT, represents the claim to both. On redemption, the buyer receives the controlling NFT and the assets move to a fresh account bound to it, where prior approvals do not apply. The seller can reclaim an unsold bundle after its deadline.

## Motivation

A marketplace can show an accurate account balance when a buyer places an order and still deliver an NFT with an empty account. Transferring the parent changes who controls the account, but an asset contract can continue to recognize spending approvals granted by the old account.

Taking custody before listing gives the buyer a specific inventory to verify. Delivering that inventory to a fresh account puts it under the redeemed NFT's control even if the seller originally held the assets in a separate wallet. A shared escrow address lets wallets and indexers find claims under the same published admission and governance policies. A seller can offer a bundle with a creator royalty and recover an unsold bundle when custody expires. Both parties can track any delivery that remains pending after exit.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Discovery

The claim collection MUST implement [ERC-721](./erc-721.md) and announce the following interface through [ERC-165](./erc-165.md). Its identifier is 0x68420221. The claim token ID MUST equal the bundle ID. Reads for known IDs MUST remain available after burn. This function MUST revert for an unknown ID.

```solidity
interface ICustodiedBundleClaim {
    function bundleClaim(uint256 bundleId) external view returns (
        address escrow, address vault, address controllingNFT,
        uint256 controllingTokenId, bytes32 manifestHash, uint8 state,
        uint256 heldMask, address seller, uint64 custodyDeadline
    );
}
```

Within a deployment on a supported chain, one identified escrow MUST coordinate the collection. Implementers MUST publish the chain ID, escrow address, asset eligibility policy and the powers of administrators and upgraders. Admission MAY restrict collections or assets. Owners meeting those conditions MUST be able to deposit without individual approval from a protocol operator. No common address across chains is required.

### Inventory

The escrow MUST expose the following reads and exits. The receipt address returned by the escrow MUST be the collection that reports it. The vault returned by the collection is the emitter of release progress for that bundle.

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

A deposit MUST take custody of the controlling ERC-721 and every manifest entry before making the claim available. Eligibility does not depend on prior custody by an account bound to the NFT. The source account itself stays at its address. The manifest MUST remain readable and immutable after deposit. It contains zero to 32 entries ordered strictly by numeric (kind, token address, token ID). Kinds 0, 1 and 2 denote [ERC-20](./erc-20.md), ERC-721 and [ERC-1155](./erc-1155.md) respectively. Amounts MUST be positive. ERC-20 entries MUST use ID zero and ERC-721 entries MUST use amount one. A manifest MUST exclude the controlling token and MUST NOT assign two kinds to the same token contract.

The commitment MUST be computed as follows using the standard Solidity ABI encoding and the entry tuple defined above. It commits to the original inventory. Delivery progress is recorded separately. Bundle identity also includes the bundle ID because repeated deposits can have the same hash.

```solidity
keccak256(abi.encode(uint256(2), block.chainid, escrow, controllingNFT, controllingTokenId, entries))
```

### Claim lifecycle

| State | Value | Action or entitlement |
|---|---:|---|
| None | 0 | No bundle |
| Deposited | 1 | Escrow holds the claim for the seller's bundle |
| Listed | 2 | Claim is available for an authorized primary sale |
| Issued | 3 | Sale transferred the claim to its holder |
| Redeemed | 4 | Buyer exit complete |
| Reclaimed | 5 | Seller exit complete |
| Redeeming | 6 | Buyer exit started and something remains in custody |
| Reclaiming | 7 | Seller exit started and something remains in custody |

A sale MUST transfer the escrow-held claim to the recipient once. Subsequent holder transfers MUST revert. Only the issued claim's current owner MAY initiate redemption. The escrow MUST burn the claim on exit and MUST prevent a second exit. An approved operator MUST NOT be able to redeem on the owner's behalf.

On redemption the escrow MUST create and initialize a fresh [ERC-6551](./erc-6551.md) account bound to the controlling NFT on this chain, using the holder's salt contribution. The controlling NFT is due to the redeemer and the entries are due to the new account. Before releasing entries to that account the vault MUST establish that the NFT reached the redeemer and the redeemer controls the account. The recovery exception below permits the redeemer to select another asset destination if delivery of the NFT is pending.

At or after the custody deadline, only the recorded seller MAY reclaim a never-issued Deposited or Listed bundle. The controlling NFT MUST return to the seller. The seller selects a nonzero destination for the entries. Protocol custody addresses MUST be rejected as destinations. The seller may select the original account. Reclaiming entries can proceed while the controlling NFT is pending.

### Partial release and events

A successful exit transaction MAY leave assets in custody. Each set bit in `heldMask` MUST identify an outstanding delivery. Bits 0 through 31 correspond to manifest indices and bit 32 to the controlling NFT. Other bits MUST be zero. A completed exit MUST have a zero mask. Burning the claim alone MUST NOT mark delivery complete.

Anyone MAY call `withdrawRemaining` with empty data to retry outstanding deliveries to the recorded destinations. The mask MUST reject bits outside the manifest and controlling NFT. Already released assets MUST NOT be delivered twice. With nonempty data encoded as abi.encode(uint8(1), destination), only the recorded NFT recipient MAY redirect outstanding manifest entries to a valid destination. During redemption this is permitted only while the controlling NFT is still held. A redirect then permits delivery of entries independently of the NFT. On reclaim the seller can redirect while entries remain. The controlling NFT destination MUST remain fixed.

The vault MUST emit the following events for the corresponding actions. `ReleaseBegun` records the initial destinations, with mode 0 for redemption and 1 for reclaim. `EntryPending` reports an attempted transfer left outstanding in a successful transaction. A gated entry may remain held without an attempted transfer. `BundleEmptied` signals the last delivery. An indexer SHOULD reconcile events against the held mask at the same block.

```solidity
event ReleaseBegun(uint256 indexed bundleId, uint8 mode, address extrasTo, address nftTo);
event EntryReleased(uint256 indexed bundleId, uint256 indexed index, address to);
event EntryPending(uint256 indexed bundleId, uint256 indexed index);
event ControllingNFTReleased(uint256 indexed bundleId, address to);
event EntriesRedirected(uint256 indexed bundleId, address to);
event BundleEmptied(uint256 indexed bundleId);
```

### Marketplace display

ERC-721 metadata SHOULD expose the chain ID, bundle ID, original NFT contract and ID, and manifest hash under properties using the keys chain_id, bundle_id, original_nft_contract, original_nft_token_id and manifest_hash. IDs SHOULD be decimal strings and the hash a 0x-prefixed 32-byte hex string. A marketplace SHOULD read the manifest at the escrow and verify its commitment before showing the assets. It SHOULD distinguish the original inventory from assets still held and label an issued claim as redeem only.

## Rationale

The receipt is the asset a marketplace already knows. A single read there supplies the escrow address and everything needed to verify the manifest or explain a pending exit. A chain-scoped hash prevents accidental reuse across escrows. Fixed manifest indices let indexers follow each delivery separately. A fresh account separates the received assets from spending permissions granted by the old account. Marketplace order formats, fee schedules and royalty routing can evolve independently.

## Backwards Compatibility

Existing ERC-721 clients can display the receipt. A client that assumes unrestricted secondary transfer will encounter a revert after issuance. The NewTrade Base receipt predates the discovery interface. Its metadata identifies the schema newtrade.bundle.v2. For that known deployment, read controller() on the receipt, cross-check receipt() on the escrow, read `getBundle(id)` and `getManifest(id)` on the escrow, and read `bundleStatus(id)` on its vault for the release record and held mask. These reads remain useful after `tokenURI` and `ownerOf` cease to resolve a burned claim. The escrow's `receiptMetadata` callback is restricted to the collection.

## Test Cases

For chain 8453, escrow address(1), controlling collection address(2), token ID 7, and one entry (0, address(3), 0, 100), the manifest hash is 0x8794c8eeb370de4efc9912f10e0782dea78bfb9ae7c0c8c36105a11acfe50a1f. The initial held mask is 0x100000001. If the NFT reaches the redeemer and the asset remains held, state is 6 and the mask is 0x1. A successful retry for bit zero gives state 4 and a zero mask. The original manifest and hash remain available throughout.

## Reference Implementation

NewTrade on Base (chain ID 8453) illustrates the custody and release semantics. Contracts source revision cd990894e8fb331d0c42e4cbb853a59a24af01f6 documents this deployment. The deployed receipt predates `bundleClaim` and its ERC-165 identifier.

| Contract | Base proxy |
|---|---|
| Escrow | 0xCc29F5c82C8b7c01Cac6036047E7413Df458f86d |
| Claim collection | 0x59900b207ea703B0bBcF8fD84C5E11dE46826185 |
| Custody vault | 0xe3a95e47B8ECC9F8E42a7BDE58dde2FBCbc69C53 |

The seller supplies the controlling NFT in both Base deposit paths. In `deposit`, the vault pulls every entry directly from the seller using token approvals. In `depositFromAccount`, all entries come from one executable account reporting the matching chain, collection and ID. This path requires enabled migration and an unused seller nonce. Each call selects one source for the entire manifest. A single deposit cannot combine wallet and account balances. Both paths lead to the same fresh-account redemption. The escrow snapshots the controlling NFT's creator royalty on listing and the wrapper itself quotes zero. The escrow emits `BundleRedeemed` or `BundleReclaimed` at exit initiation even when entries remain held.

## Security Considerations

A collection can claim any interface identifier and serve arbitrary metadata. Clients need an independent basis for trusting a deployment and the behavior of its admitted tokens. The commitment covers selected entries and quantities. Unlisted balances, future transfers and asset value fall outside it. A token can refuse delivery indefinitely. Wallets should show the pending destination and held mask, including after the receipt burns. Redirecting assets changes where recovery delivers them and deserves explicit confirmation by the beneficiary.

The Base escrow is a UUPS proxy. Administrators control fee and admission policy, and upgrade authorities can replace code governing existing custody. An interface check gives no assurance about those authorities. A controlling-collection allowlist can narrow admission to reviewed tokens. Each deployment has its own policy and trust assumptions.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

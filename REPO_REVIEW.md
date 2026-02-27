# Repository Review: naira_nft_marketPlace

## What this repository contains

This repository currently has two Solidity contracts and one incomplete test/deploy setup:

- `contracts/src/nairaNFT.sol`: ERC-721 NFT collection with pausable minting, burn support, owner mint, public paid mint, and ERC-2981 royalties.
- `contracts/src/nairaNFTMarketPlace.sol`: Marketplace for listing and buying ERC-721 tokens with a configurable marketplace fee.
- `contracts/test/nairaNFT.t.sol`: Foundry-style tests (currently non-compiling/incomplete).
- `contracts/script/deployNairaNFT.s.sol`: Foundry deploy script (currently non-compiling due to constructor mismatch).

## High-level contract behavior

### `nairaNFT`
- Inherits `ERC721`, `ERC721Pausable`, `ERC721Burnable`, `ERC2981`, `Ownable`, and `ReentrancyGuard`.
- Supports owner minting and public minting with payment checks.
- Tracks per-wallet mint counts through `mintedBy`.
- Includes `pause/unpause`, metadata base URI updates, and owner-only withdrawal.

### `NFTMarketPlace`
- Allows token owners to list approved NFTs with a fixed price.
- Buyers pay exact listing price; marketplace splits payment into fee + seller proceeds.
- Deletes listing before external calls and token transfer (reentrancy-aware order).
- Supports delisting and fee updates by the fee owner.

## Potential issues identified

1. **Compile-time bug in `nairaNFT` immutable assignment**
   - `maxSupply` is declared `immutable` and initialized inline (`= 1_000`) and then assigned again in the constructor.
   - Immutable values can only be assigned once.

2. **Deploy script cannot compile/run as written**
   - `new nairaNFT()` is called with zero constructor args, but `nairaNFT` requires 9 constructor parameters.

3. **Test file is currently non-functional**
   - Name collision: test contract declared as `contract nairaNFT is nairaNFT, Test`.
   - Constructor call mismatch (`new nairaNFT()` with no args).
   - Multiple syntax errors (`VM.expectRevert`, missing semicolons, invalid literals, undeclared `to`).
   - Test function names do not follow Foundry's `test...` convention, so even if compiled many would not execute automatically.

4. **Marketplace purchase flow can fail if listing is stale**
   - At purchase time, marketplace does not re-check ownership/approval before attempting transfer; if seller transferred/revoked approval after listing, `buy` will revert after payment transfers are attempted in the function sequence (entire tx reverts, but UX is poor and can cause unexpected failed buys).

5. **Fee configuration lacks upper bound**
   - `changeFees` only enforces `newFees > 0`; no max cap. This allows values where fees could consume nearly all sale value.

6. **Fee denomination/documentation mismatch risk**
   - Code computes fees with denominator `1000` (so `25 = 2.5%`), but some comments/docstrings describe fee as whole percentage (`5 = 5%`). This can confuse integrators.

7. **Missing repo scaffolding for Foundry project**
   - No `foundry.toml` at repository root and no dependency lock/installation metadata was found in the current snapshot, so local reproducibility is limited.

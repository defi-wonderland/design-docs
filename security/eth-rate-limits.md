# ETH Rate Limits

|                    |                             |
| ------------------ | --------------------------- |
| Author             | 0xDiscotech                 |
| Created at         | 2025-04-16                  |
| Initial Reviewers  | agusduha, skeletor-spaceman |
| Need Approval From |                             |
| Status             | In Review                   |

## Purpose

This document presents a solution to serve as the final interop safeguard against a potential bug that could lead to infinite ETH minting on L2s.

## Summary

The proposed solution is to add a rate limit to `ETH` bridged between L2s within the same interop cluster. The rate limit will be accounted for and enforced on the destination chain, ensuring that the total ETH minted per block does not exceed it.

## Problem Statement + Context

This document addresses [this issue](https://github.com/ethereum-optimism/design-docs/issues/262), which proposes adding a rate limit to the initial release as a security measure. The idea is that, in the worst-case scenario, if an invalid cross-chain message manages to mint ETH out of thin air, the damage would be limited.

This is intended to mitigate the failure mode where the batcher key is compromised and blocks are directly published to L1 with a zero-day exploit. By limiting the amount of ETH that can be minted per block, the exploit's impact is minimized. If detected quickly, the batcher key can be rotated before a significant amount of ETH is lost.

## Proposed Solution

The rate limit will be enforced in the `SuperchainETHBridge` contract via a `RATE_LIMIT` constant (so it is shared across all predeploys) and an `ethAmountPerBlock` variable.
They will be involved in two flows:

1. **In `relayETH`**:

   - If the `_amount + ethAmountPerBlock` exceeds `RATE_LIMIT`, the transaction will revert.
   - Otherwise, the `_amount` will be added to the `ethAmountPerBlock` value for the current block.

2. **In `sendETH`**:
   - If the `msg.value` exceeds `RATE_LIMIT`, the transaction will revert immediately to avoid guaranteed failure on the destination chain.

New constant and new variable (initialized when declared to save gas on the first update):

```solidity
// The maximum amount of ETH that can be minted per block
uint256 public constant RATE_LIMIT = 1000 ether;

/// @notice Amount of ETH relayed in the current block.
/// @dev Packed as [ETH amount | block number] in a single uint256.
uint256 public ethAmountPerBlock = uint128(0) | block.number;
```

The `ethAmountPerBlock` variable is packed as `[ETH amount | block number]` in a single `uint256` slot to save gas.

On `relayETH`:

```solidity
function relayETH(address _from, address _to, uint256 _amount) external {
    ...

    // Check that the rate limit is not exceeded.
    uint128 amountRelayed = uint128(ethAmountPerBlock >> 128);
    if (amountRelayed + _amount > RATE_LIMIT) revert RateLimitExceeded();

    ...

    // Update the ETH amount relayed for the current block.
    uint128 blockNumber = uint128(ethAmountPerBlock);
    if (blockNumber != block.number) {
        ethAmountPerBlock = (_amount << 128) | block.number;
    } else {
        ethAmountPerBlock += _amount;
    }

    ...
}
```

On `sendETH`:

```solidity
function sendETH(address _to, uint256 _chainId) external payable returns (bytes32 msgHash_) {
    ...

    // Ensure the amount doesn't exceed the rate limit.
    if (msg.value > RATE_LIMIT) revert RateLimitExceeded();

    ...
}
```

### Resource Usage

The gas overhead should be minimal:

- The `ethAmountPerBlock` variable is initialized to `0` amount on the current `block.number` to avoid updating the value on a zero to non-zero operation, which is more expensive.
- Adds a new constant `RATE_LIMIT`, slightly increasing deployment size.
- Introduces an additional `SLOAD` in both `sendETH` and `relayETH` for rate limit checks.
- Adds one `SSTORE` per `relayETH` execution to update `ethAmountPerBlock`, but it will be a warm write.
- Both the `block.number` and total relayed ETH are packed into a single `uint256` so only one `SSTORE` is needed to update both values.

### Single Point of Failure and Multi Client Considerations

The autorelayer must handle cases where a message fails due to exceeding the rate limit and retry it in the next block.

## Failure Mode Analysis

- Messages can fail on the destination if the total minted ETH in the block exceeds the rate limit.
- Messages may continue to fail if retried too quickly, especially when multiple chains send ETH to the same destination in a short time.

## UX Impact

This impacts capital efficiency: large ETH holders won’t be able to move large amounts in a single block. If a message fails on the destination due to the rate limit, it must wait for the next block or require a manual retry.

## Risks & Uncertainties

- How will relayers manage messages that fail due to the rate limit? Will they retry automatically? If so, how many times?
- The `sendETH` limit matches the per-block `RATE_LIMIT`. Should it be capped at a lower percentage (e.g., 90%) to avoid hitting the exact threshold and causing destination failures?
- Should very small ETH sends be disallowed in `sendETH` to avoid potential DoS vectors against relayers?

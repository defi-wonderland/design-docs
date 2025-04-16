# [Project Name]: Design Doc

|                    |                             |
| ------------------ | --------------------------- |
| Author             | 0xDiscotech                 |
| Created at         | 2025-04-16                  |
| Initial Reviewers  | agusduha, skeletor-spaceman |
| Need Approval From |                             |
| Status             | In Review                   |

## Purpose

This documents presents a solution to be used as the final interop safeguard to limit a potential bug that could lead to an infinite minting on ETH on L2s.

## Summary

The proposed solution is to add a rate limit to the `ETH` bridged between L2s forming part of the same interop cluster. The rate limit will be applied on the destination chain, accounting that the minted ETH per block cannot exceed the rate limit.

## Problem Statement + Context

The problem statement is based on the following [issue](https://github.com/ethereum-optimism/design-docs/issues/262) -- which is to include a rate limit in the initial release for the amount of ether than can be sent via cross chain sends as a security mechanism for the first release. The idea is that in the worst case, an invalid cross chain message that mints ether out of thin air would be able to do less damage.
This exists to prevent the failure mode of the batcher key being stolen and blocks being published directly to L1 that include a 0 day that is able to exploit interop. In this case, only a small amount of ether would be able to be minted before we can detect it. On detection, we will be able to swap the batcher key with only a small amount of ether being stolen.

## Proposed Solution

<!-- A high level overview of the proposed solution.
When there are multiple alternatives there should be an explanation
of why one solution was picked over other solutions.
As a rule of thumb, including code snippets (except for defining an external API)
is likely too low level. -->

The solution is to add account for the rate limit on the `SuperchainETHBridge` contract through a `RATE_LIMIT` constant (so this remains the same for the predeploy on all chains), and it will be interacted with on 2 flows:

1. Checking that the amount to relay on `relayETH` doesn't exceed the `RATE_LIMIT`:
   a. If it does, the transaction will revert.
   b. If it doesn't, the amount will be increased on the current block number
2. Checking that the amount to send on `sendETH` doesn't exceed the `RATE_LIMIT` -- otherwise the transaction will always revert on destination chain.

New constant (value TBD) and new variable:

```solidity
// The maximum amount of ETH that can be minted per block
uint256 public constant RATE_LIMIT = 1000 ether;

/// @notice Amount of ETH relayed on the current block.
/// @dev Composed of the packed values of the amount of ETH relayed and the block number.
uint256 public ethAmountPerBlock;
```

On `relayETH`:

```solidity
    function relayETH(address _from, address _to, uint256 _amount) external {
        ...

        // Make sure the rate limit is not exceeded.
        uint128 amountRelayed = uint128(ethAmountPerBlock >> 128);
        if (amountRelayed + _amount > RATE_LIMIT) revert RateLimitExceeded();


        ...

        // Update the amount of ETH relayed for the current block.
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

        // Make sure the amount to be sent is not greater than the rate limit.
        if (msg.value > RATE_LIMIT) revert RateLimitExceeded();

        ...
    }
```

### Resource Usage

The changes on resource usage are:

- Introducing a new constant `RATE_LIMIT` that will slightly increase deployment size.
- Adding one check on `sendETH` to ensure that the amount to be sent doesn't exceed the rate limit -- which includes one more `SLOAD` operation.
- Adding one check on `relayETH` to ensure that the amount to be relayed doesn't exceed the rate limit -- which includes one more `SLOAD` operation.
- Updating the `ethAmountPerBlock` variable on each `relayETH` and `sendETH` call, which includes one more `SSTORE` operation -- however, this will be an update over a non-zero value, so it shouldn't impact too much the gas usage.

Both the current `block.number` and the `ethAmountPerBlock` are packed into a single `uint256` variable to make it more efficient by storing them in one slot -- reserving 128bits for each value.

### Single Point of Failure and Multi Client Considerations

The autorelayer will need to handle the case where the message fails due to exceeding the rate limit and retry it on a new block.

## Failure Mode Analysis

Messages can fail on the destination if the total minted ETH in the block exceeds the rate limit.

Messages may continue to fail if retried too quickly, especially when multiple chains send ETH to the same destination in a short time.

## UX Impact

This is a great impact to capital efficiency, as it will prevent large ETH holders to move large amounts of ETH in a single block. It becomes worse if the message fails on destination chain due to exceeding the rate limit, needing to wait for it to be included in a new block or requiring a manual retry.

## Risks & Uncertainties

- How will the relayer handle the messages that failed due to the rate limit? Will it try to relay them again? How many times?
- The limit amount on `sendETH` is the total amount of ETH that can be sent on a single block. Should this get limited to a lower amount (e.g. 90% of the rate limit) to prevent failures on destination chain?
- Should be prevented users from sending too little ETH amount on `sendETH` to avoid a DoS on the autorelayer?

```

```

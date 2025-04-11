# Forked Testing Framework for L2 Networks: Design Doc

## Purpose

This document outlines the proposed design for an integrated L2 Forked Test suite and defines the scope of this integration.

## Summary

This proposal describes a testing framework for L2 networks that lets us run tests against realistic contract states. The framework applies Network Upgrade Transactions (NUTs) to forked networks, creating a consistent environment for testing Predeploy contracts. By implementing NUTs in Solidity or combining Go with Solidity, we can improve test coverage, build confidence in our upgrades, and identify issues earlier.

## Problem Statement + Context

As Predeploy contracts development advances, we currently lack an efficient method to run test suites against the state of arbitrary networks. We need to develop a configurable testing framework that accurately replicates the state of Predeploy contracts within selected networks, initialized from a specified block number (initial state).
This framework should be able to take a network's initial state and apply the required Network Upgrade Transactions (NUTs) to reach the latest version of the contracts for running the test suite. This approach would help increase confidence that the upgrade process doesn't introduce unexpected bugs, allowing contributors to catch errors earlier in the release process. While this implementation won't perfectly mirror the environment after NUTs are applied to the real network, it represents a significant improvement to our testing capabilities.
The solution must provide a deterministic state to the test suite, enabling the execution of targeted test cases against this final state. This state results from applying a set of NUTs to the initial forked state. Therefore, we need to implement a structured way to describe these NUTs and a method to apply them.
It's also important that this solution remains loosely coupled with the test setup itself, so that running tests against different chains starting with different initial states requires minimal changes to the Setup test file.

## Proposed Solution

Currently, L1 forked state is abstracted by ForkLive, which handles reading the L1 contract's code and upgrading when needed. Ideally, we'd like a similar approach for L2 fork tests, where a single contract is responsible for setting up L2 Predeploys in the final state needed for our test suite. We propose a two-part approach: one part abstracts the NUTs that occur in each upgrade, while the other part executes them sequentially. For L2, we have the advantage of knowing the addresses in advance, so unlike ForkLive, storing addresses isn't necessary.

### Implementation Components

1. **NUT Definitions**: Specifications of the Network Upgrade Tests to be executed prior to test suite initialization

```solidity
interface NUTExecutor {
    function execute(bytes calldata _calldata) external;
}

contract XForkExecutor is NUTExecutor {
    function execute(bytes calldata _calldata) external {
        ///         1. Deploy a new `L1BlockImpl` contract.
        ///         2. Upgrade only the `L1Block` contract to the new implementation by
        ///            calling `L2ProxyAdmin.upgrade(address(L1BlockProxy), address(L1BlockImpl))`.
        ///         3. Call `L1Block.setXFork()` to pull the values from L2 contracts.
        ///         4. Upgrades the remainder of the L2 contracts via `L2ProxyAdmin.upgrade()`.
    }
}

```

1. **Execution Script**: A specialized module that manages the sequential execution of defined NUTs

```solidity
contract Upgrader {
    function upgrade(NUTExecutor[] memory _executors) external {
	    // Apply NUTExecutors based on the initial state
    }
}

```

This approach facilitates the specification of NUT sets to be executed from a defined initial state. The implementation would be encapsulated within a contract similar to ForkLive and serve as an alternative to the L2Genesis script, to be invoked during the Setup phase.

### Integration with Client Upgrade Process

Currently, we use Go scripts to prepare NUTs for each upgrade, which are then passed back to the pipeline. These NUTs are defined in dedicated Go files that build them individually. Ideally, we should share these transaction definitions between the client and our test suite to eliminate code duplication and ensure that our test upgrades closely mirror production environments.

Once we have contracts that handle entire sets of NUTs, the transactions in the Go scripts can be replaced by two standard transactions:

1. Deploying the appropriate NUTExecutor contract for that particular upgrade
2. Calling the execute function on that contract

This approach offers flexibility and power as the exact same transactions could be executed both in production upgrades and our Solidity test suite (provided we use the same arguments). Additionally, it would enable the test suite to easily fuzz transactions as needed.

**Advantages:**

- Provides high-fidelity representation of Predeploy states by simulating complete upgrade paths from an initial network state
- Offers flexible configuration options for various testing scenarios

However, current limitations prevent us from implementing a truly shared approach, as we require NUT scripts to execute within the context of privileged accounts such as ProxyAdmin or the Depositor Account. To address this challenge, we would need to implement one of the following solutions:

#### Pectra Hardfork and EIP-7702

The Ethereum Pectra Hardfork introduces EIP-7702, which enables EOAs to delegate control to smart contracts capable of executing code directly from the address. This would allow us to integrate the NUT Executor approach into L2 upgrades, as privileged accounts could execute NUT scripts via delegated calls.

#### ProxyAdmin Upgrade

Alternatively, we could upgrade the L2 ProxyAdmin contract to permit limited delegated calls, enabling NUT scripts to execute within the ProxyAdmin context and perform contract upgrades as necessary.

```solidity
contract L2ProxyAdmin is ProxyAdmin {

    /// @dev allow Constants.DEPOSITOR_ACCOUNT to perform delegated calls
    function _checkOwner() internal view override {
        require(owner() == _msgSender() || Constants.DEPOSITOR_ACCOUNT == _msgSender(), "Ownable: caller is not the owner");
    }

	function performDelegateCall(address _target) external payable onlyOwner {
        (bool success,) = _target.delegatecall(abi.encodeCall(INUTExecutor.execute, ()));
        require(success, "ProxyAdmin: delegatecall to target failed");
	}
}

```

> Designating the Depositor Account as the owner of the L2ProxyAdmin would allow for the execution of all NUTs by impersonating only a single EOA.

## Impact on Developer Experience

Moving away from Go in favor of Solidity to define the NUTs would imply a change in the upgrade release process. Initially, this transition would require updating the Go scripts and creating Solidity NUT definitions. However, the result would be a more cohesive and integrated process for upgrading L2 Predeploys. This approach would also increase confidence in our test suite by ensuring it reflects a more realistic state of the network.

## Alternative Approach

Network upgrade transactions are defined in Go scripts following the naming convention <fork_name>\_upgrade_transactions.go (e.g., fjord_upgrade_transactions.go). These scripts are used by the client to send the necessary transactions.
We could create middleware Go scripts that consume these transactions and return them in a format which can be consumed by Solidity for execution from Foundry tests. The L2ForkLive would call these Go scripts via FFI and parse the response for it to execute each one of the calls.

**Advantages:**

- No changes needed to existing Go scripts
- Maintains consistency between NUTs used in testing and actual upgrades

**Disadvantages:**

- Requires parsing of complex data returned from Go scripts
- Introduces middleware that's tightly coupled with the L2ForkLive Foundry script

## Risks & Uncertainties

- Are there any issues by having differences between implementation contract addresses and the expected derived address from the `Predeploys.predeployToCodeNamespace` function?

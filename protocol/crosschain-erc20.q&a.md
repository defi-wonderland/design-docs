[Q1:](https://github.com/ethereum-optimism/design-docs/pull/203#discussion_r1966153251)

    What if the non xERC20 token was already bridged to another chain by a different bridging method? Would you then need the Lockbox on both chains, not just the hub chain?

Ideally, the Lockbox should be on a single chain or hub chain to avoid liquidity issues with the Lockbox. If there are different methods to bridge the tokens, this could cause a situation where, on a given chain, there are not enough funds in the Lockbox to unlock the original ERC20. In this case, the funds should be returned using the bridge initially used and locked in the hub chain.

[Q2:](https://github.com/ethereum-optimism/design-docs/pull/203#discussion_r1966256013)

    So tokens do not need to be deterministically deployed with the adaptor? How is the total supply of a token across all the chains calculated? Does this make it harder?

In this case, the bridge (or SuperchainTokenBridge) calls the Adapter and not the Token contract. For this reason, it is not necessary for the Tokens to have the same address, but the Adapters must.
To calculate liquidity: If the tokens have the same address across all chains, the usual methods would be used. However, if the tokens do not have the same address, the variable in the adapter that stores the adapted token's address could be used to redirect the calculation.

[Q3:](https://github.com/ethereum-optimism/design-docs/pull/203#discussion_r1966266088)

    if the xerc20 contract is passed into the constructor of the adaptor, wouldn't the xerc20 address still need to be the same across all chains in order for the adaptor address to be the same across all chains?

[Self-Answer:](https://github.com/ethereum-optimism/design-docs/pull/203#discussion_r1966273679)
oh I see the factory uses CREATE3 so constructor params won't matter

[Q4:](https://github.com/ethereum-optimism/design-docs/pull/203#discussion_r1966276578)

    is this one Lockbox per token?
    same with the adaptor? one adaptor per xerc20 per chain?

Yes, it is one lockbox or adapter per token. While it is possible for the token issuer to deploy an arbitrary number of lockboxes and adapters and grant them mint/burn permissions, this would not be ideal and would be a mistake on the part of the token issuer.

[Q5:](https://github.com/ethereum-optimism/design-docs/pull/203#discussion_r1966281933)

    how common is it for xERC20 addresses to be the same?

In this case, we are requiring as a prerequisite that the token be deployed at the same address. If this is not the case, the token issuer would need to resolve this situation, as it is outside the scope of this solution.

[Q6:](https://github.com/ethereum-optimism/design-docs/pull/203#discussion_r1966286737)

    Not sure I understand this one: so the CrosschainERC20Converter contract is a CrosschainERC20 contract? What does the undo call do?

The CrosschainERC20Converter would be a CrosschainERC20 with an extension that allows burning xERC20 and minting CrosschainERC20 within the same contract. It is like a Lockbox, but instead of locking, it burns the xERC20. The undo function would be the inverse of convert, burning CrosschainERC20 and minting the xERC20 again.

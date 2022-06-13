---
slug: erc4626-usage
title: "Introducing Apollo: analyzing the usage of ERC4626 across chains"
authors: [jonas, francesco]
tags: [apollo, erc4626, tutorial]
---

## Introduction
For the last few months, we've been workig on [Apollo](https://github.com/chainbound/apollo) 
([docs](https://apollo.chainbound.io)), a tool
to make it easy for anyone to **query**, **filter**, **transform** and **save** EVM chaindata based on a schema.

We built Apollo because we needed to be able to scrape obscure EVM chaindata fast, and the tools currently out there
were too limiting in what they could do. They either run only on a couple chains, or rely on indexing, which is a process
that takes time and is not feasable for just any protocol. **Apollo** interacts directly with the standardized
[JSON RPC API](https://eth.wiki/json-rpc/API) that any EVM node implementation should expose. This means that
as long as you have a JSON RPC API for your chain, **Apollo will be able to run there**. The only thing
you need is an [ABI](https://www.quicknode.com/guides/solidity/what-is-an-abi).

We believe that the best way to introduce a tool like this is to show its value in a **real world example**.
Inspired by [this tweet](https://twitter.com/boredGenius/status/1533531858591309824), we set about analyzing
the usage of the new [ERC4626 Tokenized Vault Standard](https://eips.ethereum.org/EIPS/eip-4626) across
multiple chains. If you would like to explore more chains, at the
end of the article you will be able to do that yourself, because **Apollo will be open source**.

## What is ERC4626?
ERC4626 is a new token standard aims to clear up the problems with having different implementations of tokenized vaults.
One of the most powerful mechanisms in DeFi is **composability**, but composability doesn't work without standards. If you know 
[ERC20](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) (token standard) or 
[ERC721](https://ethereum.org/en/developers/docs/standards/tokens/erc-721/) (NFT standard), you know how crucial they are. One of the more important products in DeFi are yield-bearing tokenized
vaults. If you've ever staked Sushi in return for xSushi, or deposited ETH on Aave and received aETH, you've used these products.
They represent shares of an underlying token that generate interest over time.
The problem is that when building applications that can integrate with these tokens, you have to build an integration for each
separate implementation. This is complex, error-prone, and resource intensive, resulting in less applications actually doing it. **Bad for composability**.

ERC4626 aims to set a standard for these products called the Tokenized Vault Standard. 
If an application works with ERC4626, it works with any yield-bearing token that 
implements the standard. This will drastically lower the integration effort and will enable a renaissance in DeFi.
It will, for example, provide lending platforms the ability to easily accept any ERC4626 token as collateral,
which would be one example of **composable yield**.

Let's take a look at the interface, because we'll be needing that later:
```sol
abstract contract IERC4626 is ERC20 {
    /*////////////////////////////////////////////////////////
                      Events
    ////////////////////////////////////////////////////////*/

    /// @notice `sender` has exchanged `assets` for `shares`,
    /// and transferred those `shares` to `receiver`.
    event Deposit(address indexed sender, address indexed receiver, uint256 assets, uint256 shares);

    /// @notice `sender` has exchanged `shares` for `assets`,
    /// and transferred those `assets` to `receiver`.
    event Withdraw(address indexed sender, address indexed receiver, uint256 assets, uint256 shares);

    /*////////////////////////////////////////////////////////
                      Vault properties
    ////////////////////////////////////////////////////////*/

    /// @notice The address of the underlying ERC20 token used for
    /// the Vault for accounting, depositing, and withdrawing.
    function asset() external view virtual returns (address asset);

    /// @notice Total amount of the underlying asset that
    /// is "managed" by Vault.
    function totalAssets() external view virtual returns (uint256 totalAssets);

    /*////////////////////////////////////////////////////////
                      Deposit/Withdrawal Logic
    ////////////////////////////////////////////////////////*/

    /// @notice Mints `shares` Vault shares to `receiver` by
    /// depositing exactly `assets` of underlying tokens.
    function deposit(uint256 assets, address receiver) external virtual returns (uint256 shares);

    /// @notice Mints exactly `shares` Vault shares to `receiver`
    /// by depositing `assets` of underlying tokens.
    function mint(uint256 shares, address receiver) external virtual returns (uint256 assets);

    /// @notice Redeems `shares` from `owner` and sends `assets`
    /// of underlying tokens to `receiver`.
    function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) external virtual returns (uint256 shares);

    /// @notice Redeems `shares` from `owner` and sends `assets`
    /// of underlying tokens to `receiver`.
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) external virtual returns (uint256 assets);

    /*////////////////////////////////////////////////////////
                      Vault Accounting Logic
    ////////////////////////////////////////////////////////*/

    /// @notice The amount of shares that the vault would
    /// exchange for the amount of assets provided, in an
    /// ideal scenario where all the conditions are met.
    function convertToShares(uint256 assets) external view virtual returns (uint256 shares);

    /// @notice The amount of assets that the vault would
    /// exchange for the amount of shares provided, in an
    /// ideal scenario where all the conditions are met.
    function convertToAssets(uint256 shares) external view virtual returns (uint256 assets);

    /// @notice Total number of underlying assets that can
    /// be deposited by `owner` into the Vault, where `owner`
    /// corresponds to the input parameter `receiver` of a
    /// `deposit` call.
    function maxDeposit(address owner) external view virtual returns (uint256 maxAssets);

    /// @notice Allows an on-chain or off-chain user to simulate
    /// the effects of their deposit at the current block, given
    /// current on-chain conditions.
    function previewDeposit(uint256 assets) external view virtual returns (uint256 shares);

    /// @notice Total number of underlying shares that can be minted
    /// for `owner`, where `owner` corresponds to the input
    /// parameter `receiver` of a `mint` call.
    function maxMint(address owner) external view virtual returns (uint256 maxShares);

    /// @notice Allows an on-chain or off-chain user to simulate
    /// the effects of their mint at the current block, given
    /// current on-chain conditions.
    function previewMint(uint256 shares) external view virtual returns (uint256 assets);

    /// @notice Total number of underlying assets that can be
    /// withdrawn from the Vault by `owner`, where `owner`
    /// corresponds to the input parameter of a `withdraw` call.
    function maxWithdraw(address owner) external view virtual returns (uint256 maxAssets);

    /// @notice Allows an on-chain or off-chain user to simulate
    /// the effects of their withdrawal at the current block,
    /// given current on-chain conditions.
    function previewWithdraw(uint256 assets) external view virtual returns (uint256 shares);

    /// @notice Total number of underlying shares that can be
    /// redeemed from the Vault by `owner`, where `owner` corresponds
    /// to the input parameter of a `redeem` call.
    function maxRedeem(address owner) external view virtual returns (uint256 maxShares);

    /// @notice Allows an on-chain or off-chain user to simulate
    /// the effects of their redeemption at the current block,
    /// given current on-chain conditions.
    function previewRedeem(uint256 shares) external view virtual returns (uint256 assets);
}
```

## Analysis
Since we believe ERC4626 will be a very important building block in future DeFi protocols, we wanted to analyze its usage.
Partly because we live in a multi-chain world, and partly because we want to highlight a competitive feature that **Apollo** has,
we will be looking at 4 chains: **Ethereum**, **Polygon**, **Arbitrum** and **Optimism**. This analysis is also meant to 
give you an introduction as to how Apollo can be used.

### Collecting the data
This is the game plan: we are going to collect every ERC4626 `Deposit` event across
the chains, Our research period starts February 1, and ends June 13. 


## References
1. https://decrypt.co/99695/how-erc-4626-could-fuel-next-wave-of-defi
---
slug: erc4626-usage
title: "Introducing Apollo: analyzing usage of ERC4626 across chains"
authors: [jonas, francesco]
tags: [apollo, erc4626, tutorial]
---

## Introduction
For the last few months, we've been workig on [Apollo](https://github.com/chainbound/apollo) ([docs](/)), a tool
to make it easy for anyone to **query**, **filter**, **transform** and **save** EVM chaindata based on a schema.

We built Apollo because we needed to be able to scrape obscure EVM chaindata fast, and the tools currently out there
were too limiting in what they could do. They either run only on a couple chains, or rely on indexing, which is a process
that takes time and is not feasable for just any protocol. **Apollo** interacts directly with the standardized
[JSON RPC API](https://eth.wiki/json-rpc/API) that any EVM node implementation should expose. This means that
as long as you have a JSON RPC API for your chain, *Apollo will be able to run there*. The only thing
you need is an [ABI](https://www.quicknode.com/guides/solidity/what-is-an-abi).

We believe that the best way to introduce a tool like this is to show its value in a **real world example**.
Inspired by [this tweet](https://twitter.com/boredGenius/status/1533531858591309824), we set about analyzing
the usage of the new [ERC4626 Tokenized Vault Standard](https://eips.ethereum.org/EIPS/eip-4626) across
**Ethereum**, **Arbitrum**, and **Optimism**. If you would like to explore more chains, you will soon be able to
do that yourself, because **Apollo will be open source**.

## What is ERC4626?
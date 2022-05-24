---
sidebar_position: 1
slug: /
---
# Introduction
**Collecting blockchain data is no easy task**. As DeFi researchers, we've spent countless hours building infrastructure 
that would allow us to get the data we need for a specific application: generating ABI bindings with 
[Typechain](https://github.com/dethcrypto/TypeChain) or [abigen](https://geth.ethereum.org/docs/dapp/native-bindings), 
building the application logic for fetching historical data, transforming and parsing the raw call results, filtering logs, creating SQL tables for saving that data...

We've built `apollo` to change and simplify all that. With `apollo`, you can **query**, **filter**, **transform** and **save** 
EVM **chaindata** based on a [schema](./category/schema) (more on that later). 
When we talk about **chaindata**, what we mean is one of the following things:

* **Contract methods**
* **Events**
* **Transactions**
* **Native Balances**


`apollo` has 2 main modes:
* **Historical mode**: ideal for fetching historical data and building datasets. Based on a specified time range, `apollo`
will process your schema with these historical states, and build up big datasets in no time.
* **Real-time mode**: `apollo` can also work in real-time. For **methods** and **balances** this would mean querying 
the latest state every `x` seconds, which could be useful for dashboards or other time-sensitive products. 
For **events** and **transactions**, we provide real-time feeds which can be used for alerts, mempool monitoring, and anything
else you can come up with.

:::info
For now, `apollo` only works with EVM-based chains. This will change in the future.
:::
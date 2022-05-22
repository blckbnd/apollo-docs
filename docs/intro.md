---
sidebar_position: 1
slug: /
---
# Introduction
Collecting blockchain data is no easy task. `apollo` aims to change that.

`apollo` is a program for querying and collecting EVM chaindata based on a [schema](./category/schema/). Chaindata in this case is
one of these things:

* **Contract methods**
* **Events**
* **Transactions**
* **Balances**

It can run in 2 modes:
* **Historical mode**: here we define a block range and interval for method calls, or just a block range for events. `apollo`
will run through all the blocks and collect the data we've defined in the schema. This can be useful for backtesting etc.
* **Realtime mode**: we define a time interval at which we want to collect data. This is useful for building a backend database
for stuff like dashboards for live strategies.

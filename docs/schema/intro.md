# Introduction
The schema is in the form of a DSL implemented with [HCL](https://github.com/hashicorp/hcl) to define the data
we're interested in. This means that basic arithmetic operations and ternary operators
for control flow are supported by default. 

## Variables
We can start by defining variables in a `variables` map. For example:
```hcl
variables = {
  weth_address = "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"
}
```
These variables will be available for use across the rest of our schema.

## Query
The next top-level element is always a `query` block, along with the name of the query.
This name will be used to determine the name of the output file or the name of the table in a database.
For example:
```hcl
query usdc_eth_swaps {
  chain = "ethereum"

  ...
}
```
This `query` with CSV output enabled would create `usdc_eth_swaps.csv`, and with database output we would get the
`usdc_eth_swaps` table.

In the `query` block, we must define the `chain`. This will determine the RPC for the query, and your chain name
must correspond to a chain in the configuration file (`config.yml`). **For now, only one query at a time is supported**.
Later, you will be able to execute multiple queries over multiple chains at the same time, for data that has
cross-chain relevance.

Everything in a `query` block needs to happen at the same time. This means that you can't call a method **and**
filter for events in one query (unless you're calling a method when a certain event occurs).

This is where things become interesting. Inside of the `query` block we can choose between a couple of options: **global events**
and **contracts**. Let's start with the first.

### Global Event
If we want to query certain events that are not tied to a specific address, we create an `event` block inside of the `query` block.
For example:
```hcl
query all_arbitrum_transfers {
  chain = "arbitrum"

  event Transfer {
    abi = "erc20.abi.json"

    ...
  }
}
```
This query would look for all of the ERC20 `Transfer` events that happen on the **Arbitrum** rollup. Since `Transfer` is
defined in the ERC20 ABI, we need a property inside `event` to specify the ABI.

### Contract
The other option is a `contract` block. This allows you to **call methods** or **filter events** that are tied to a specific
contract. For example:
```hcl
query usdc_eth_reserves {
  chain = "arbitrum"

  contract "0x905dfCD5649217c42684f23958568e533C711Aa3" {
    abi = "unipair.abi.json"

    ...
  }
}
```
This in itself is not a very useful example, let's continue with **calling methods**.
### Contract Methods
Building on the last example, let's see how we can call `getReserves` and define which outputs we want for further processing:
```hcl
query usdc_eth_reserves {
  chain = "arbitrum"

  contract "0x905dfCD5649217c42684f23958568e533C711Aa3" {
    abi = "unipair.abi.json"

    method getReserves {
      outputs = ["_reserve0", "_reserve1"]
    }

    ...
  }
}
```
The method name `getReserves` and the output names `_reserve0` and `_reserve1` are exactly how they are defined in the ABI, 
so be careful with typos. This method has one more output (`_blockTimestampLast`), but since we're not interested in that,
we can just omit it.

### Contract Events
Contract level events are almost the same as global events, but there's no need for an `abi` property. For example:
```hcl
query usdc_eth_swaps {
  chain = "arbitrum"

  contract "0x905dfCD5649217c42684f23958568e533C711Aa3" {
    abi = "unipair.abi.json"

    event Swap {
      outputs = ["amount1In", "amount0Out", "amount0In", "amount1Out"]
    }

    ...
  }
}
```
Would allow you to look at all the swaps on the USDC-ETH SushiSwap pool on Arbitrum, and save the outputs for 
further processing.

## Processing Data
We've talked a lot about **how** we're supposed to get our data, now let's see **what** we can do with it.
### Transform
`contract` and global `event` blocks can have additional `transform` blocks. These blocks can work with previously fetched
data to do some transformations, before proceeding to the final output. I think this can best be explained with an example.
In the previous example where we're recording `Swap` events on the USDC-WETH pool, the outputs we're getting are very raw.
For one, they're formatted according to the number of decimals of the token, which is not a great format to work with.
Let's say we want to change that:
```hcl
query usdc_eth_swaps {
  chain = "arbitrum"

  contract "0x905dfCD5649217c42684f23958568e533C711Aa3" {
    abi = "unipair.abi.json"

    event Swap {
      outputs = ["amount1In", "amount0Out", "amount0In", "amount1Out"]
    }

    transform {
      usdc_sold = parse_decimals(amount1In, 6)
      eth_sold = parse_decimals(amount0In, 18)

      usdc_bought = parse_decimals(amount1Out, 6)
      eth_bought = parse_decimals(amount0Out, 18)

      buy = amount0Out != 0
    }
  }

  ...
}
```
The `apollo` DSL provides some functions to work with, and in this case we can use `parse_decimals` to do an initial clean up
of the data. We save the results in some easy-to-remember variables that we can use later. The number `1` in the outputs
is always USDC, which has 6 decimals. Anything with the number `0` is WETH, with 18 decimals.

We can also define a `buy` boolean, which will keep track of wether the swap bought or sold WETH.

### Save (TODO)
Any `input` or `output` is provided as a variable by default.
Other variables available are:
* `timestamp`
* `blocknumber`
* `contract_address`

And for `events`:
* `tx_hash`

The available functions are:
* `lower(str)`
* `upper(str)`
* `parse_decimals(raw, decimals)`

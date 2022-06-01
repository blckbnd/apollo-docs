# Schema examples
Here you can find some example schemas to get you started.

## Events
:::info
When filtering events, **start** and **end** parameters are required. These schema's could start with
```hcl
start_time = format_date("02-01-2006 15:04", "01-05-2022 00:00")
end_time = format_date("02-01-2006 15:04", "31-05-2022 00:00")
```
or 
```hcl
start_block = 13400000
end_block = 13500000
```
:::
### Uniswap V2 ETH-USDC swap price vs. mid price
This example gets the swap price of every Swap event on the ETH-USDC
Sushiswap pool. It also fetches the reserves at the previous block, to
calculate the mid price before the swap. Note that events are at the transaction level,
while calling methods is at the block level (the state we get is the state at the end
of the target block).
```hcl
// query defines the name of your query -> name of your output files and SQL tables
query usdc_eth_swaps {
  chain = "arbitrum"

  contract {
    // specify the contract address
    address = "0x905dfCD5649217c42684f23958568e533C711Aa3" 
    // contract needs an ABI
    abi = "unipair.abi.json"

    // Listen for events
    event Swap {
      // The outputs we're interested in, same way as with methods.
      outputs = ["amount1In", "amount0Out", "amount0In", "amount1Out"]

      // We can call methods on the occurence of the Swap event
      method getReserves {
        // Call at the previous block
        block_offset = -1
        // These are the outputs we're interested in. They are available 
        // to transform as variables in the "transform" block below. Outputs should
        // be provided as a list.
        outputs = ["_reserve0", "_reserve1"]
      }
    }

    // "transform" blocks are at the contract-level,
    // they allow you to do some preliminary cleaning of the outputs
    // before moving on.
    transform {
      eth_reserve = parse_decimals(_reserve0, 18)
      usdc_reserve = parse_decimals(_reserve1, 6)

      usdc_sold = parse_decimals(amount1In, 6)
      eth_sold = parse_decimals(amount0In, 18)

      usdc_bought = parse_decimals(amount1Out, 6)
      eth_bought = parse_decimals(amount0Out, 18)

      buy = amount0Out != 0
    }
  }

  // Besides the normal context, the "save" block for events provides an additional
  // variable "tx_hash". "save" blocks are at the query-level and have access to variables
  // defined in the "transform" block
  save {
    timestamp = timestamp
    block = blocknumber
    contract = contract_address
    tx_hash = tx_hash

    eth_reserve = eth_reserve
    usdc_reserve = usdc_reserve
    mid_price = usdc_reserve / eth_reserve

    // Example: we want to calculate the price of the swap.
    // We have to make sure we don't divide by 0, so we use the ternary operator.
    swap_price = eth_bought != 0 ? (usdc_sold / eth_bought) : (usdc_bought / eth_sold)
    direction = buy ? b : s
    size_in_udsc = eth_bought != 0 ? usdc_sold : usdc_bought
  }
}
```
### Per-swap fees on Uniswap V3
In this example we calculate every swap fee in USDC on the Uniswap V3 WETH-USDC pool (0.3% fee tier).
We do this on Ethereum mainnet. We also note whether the swap bought of sold ETH.
```hcl
query "usdc_eth_fees_3000" {
  chain = "ethereum"

  contract {
    address = "0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D8" 
    abi = "uni_v3_pool.abi.json"

    event "Swap" {
      outputs = ["sender", "recipient", "amount0", "amount1", "liquidity", "tick"]
    }

    transform {
      eth_amount = parse_decimals(amount1, 18)
      usdc_amount = parse_decimals(amount0, 6)
    }
  }

  save {
    timestamp = timestamp
    block = blocknumber
    tx = tx_hash

    eth_amount = eth_amount
    usdc_amount = usdc_amount
    liquidity = liquidity
    tick = tick

    sender = sender
    recipient = recipient

    // Calculate the swap fee in USDC (0.3% fee tier)
    lp_fee = abs(0.003 * usdc_amount)

    type = eth_amount < 0 ? "ETH_SELL" : "ETH_BUY"
  }
}
```
### Record every ERC20 transfer on Ethereum and Arbitrum
This is an example of where a loop would come in handy. We also call `decimals()` on the contract
because we want to save the parsed value, and we filter out `0` values. Note that in this case
it's recommended to save the `chain`, since you otherwise won't be able to determine on which chain the
transfer took place.
```hcl
loop {
  items = ["arbitrum", "ethereum"]

  query arbi_eth_transfers {
    chain = item

    event Transfer {
      abi = "erc20.abi.json"

      outputs = ["from", "to", "value"]

      method decimals {
        outputs = ["decimals"]
      }

      transform {
        value = parse_decimals(value, decimals)
      }
    }

    filter = [
      value != 0
    ]

    save {
      chain = chain
      block = blocknumber
      timestamp = timestamp
      tx_hash = tx_hash

      from = from
      to = to
      value = value
    }
  }
}
```
## Methods
:::info
When calling methods, an **interval**, along with **start** and **end** parameters is mandatory, otherwise
`apollo` won't know when to call the methods. An example start of the following schema's would be
```hcl
start_time = format_date("02-01-2006 15:04", "01-06-2022 16:10")
end_time = now
// Call method every 30s. This will be converted to an estimated block interval.
time_interval = 30
```
or
```hcl
start_block = 13400000
end_block = 13500000
// Call every 100 blocks.
block_interval = 100
```
:::
### Mid price of Uniswap V2 pools
This example shows you how to calculate the mid price of a Uniswap V2 pool using the reserves.
We do this for the WETH-USDC pool on Arbitrum.
```hcl
query usdc_eth_mid_price {
  chain = "arbitrum"

  contract {
    address = "0x905dfCD5649217c42684f23958568e533C711Aa3" 
    abi = "unipair.abi.json"

    // Call methods
    method getReserves {
      // These are the outputs we're interested in. They are available 
      // to transform as variables in the "save" block below. Outputs should
      // be provided as a list.
      outputs = ["_reserve0", "_reserve1"]
    }


    transform {
      eth_reserve = parse_decimals(_reserve0, 18)
      usdc_reserve = parse_decimals(_reserve1, 6)
    }
  }

  save {
    block = blocknumber
    timestamp = timestamp
    mid_price = usdc_reserve / eth_reserve
  }
}
```
### Query token balance
In this example we first define some variabels for later use, and then query our USDC balance
on Ethereum mainnet. The custom function `token_balance(owner, token)` returns the parsed balance.
```hcl
// Define variables for later use
variables = {
  owner = "0xDeAdB33F"
  usdc = "0xFF970A61A04b1cA14834A43f5dE4533eBDDB5CC8"
}

query usdc_balance {
  chain = "ethereum"

  // Previous blocks are optional, we can just skip straight to the save block
  save {
    block = blocknumber
    time = timestamp
    account = owner
    // Custom helper function which returns the parsed ERC20 balance
    account_balance = token_balance(owner, usdc)
  }
}
```
### Query native balance
If you want to query your native balance on a specific chain, let's say `avax`, you would use the
`balance(owner)` helper function.
```hcl
query avax_balance {
  chain = "avax"

  save {
    block = blocknumber
    time = timestamp

    account_balance = balance("0xDeAdB33F")
  }
}
```
# Schema Examples
Here you can find some example schemas to get you started.

## Events
### ETH-USDC swap price vs. mid price
This example gets the swap price of every Swap event on the ETH-USDC
Sushiswap pool. It also fetches the reserves at the previous block, to
calculate the mid price before the swap. Note that events are at the transaction level,
while calling methods is at the block level (the state we get is the state at the end
of the target block).
```hcl
// query defines the name of your query -> name of your output files and SQL tables
query usdc_eth_swaps {
  chain = "arbitrum"

  // specify the contract address
  contract "0x905dfCD5649217c42684f23958568e533C711Aa3" {
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
### Record every ERC20 transfer, and parse the value
```hcl
query arbitrum_transfers {
  chain = "arbitrum"

  event Transfer {
    abi = "erc20.abi.json"

    outputs = [
      "from",
      "to",
      "value"
    ]

    // Because it could be any ERC20 transfer, we don't
    // know the decimals in advance and need to call them.
    // The code would somehow need to know that it should call this on
    // the contract that emitted the event.
    method decimals {
      outputs = ["decimals"]
    }


    transform {
      amount_parsed = parse_decimals(value, decimals)
    }
  }

  save {
    sender = from
    receiver = to
    amount = amount_parsed
  }
}
```
## Methods
### Calculate the mid price of a Uniswap V2 pool
```hcl
query usdc_eth_mid_price {
  chain = "arbitrum"

  contract "0x905dfCD5649217c42684f23958568e533C711Aa3" {
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
### Get your USDC balance over a period of time
```hcl
query usdc_balance {
  chain = "ethereum"
  
  contract usdc_balance "0xFF970A61A04b1cA14834A43f5dE4533eBDDB5CC8" {
    abi = "erc20.abi.json"

    method balanceOf {
      // Inputs should be defined as a map.
      inputs = {
        address = "0xD3ADBEEF"
      }

      outputs = ["balance"]
    }

  }
  save {
    time = timestamp
    account = address
    account_balance = parse_decimals(balance, 18)
  }
}
```
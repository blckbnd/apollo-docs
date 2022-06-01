---
sidebar_position: 3
---
# Advanced features
## Methods in events
For some use cases it could be helpful to be able to call a method when a certain event occurs.
One use case is recording all erc20 `Transfer` events, and wanting to parse the value using the decimals.
We don't know the decimals in advance, so we can call `decimals()` at the time the event occurs. This is the syntax:
```hcl
query eth_transfers {
  chain = "ethereum"

  event Transfer {
    abi = "erc20.abi.json"

    outputs = ["from", "to", "value"]

    // Will automatically know to call the method on the
    // event address
    method decimals {
      outputs = ["decimals"]
    }

    transform {
      value = parse_decimals(value, decimals)
    }
  }

  save {
    ...
  }
```
:::note
Since `decimals` stay the same per token, the result is cached per token, resulting in a lot less calls
for schema's like this.
:::
## Loops
Sometimes, you will want to execute a query for a lot of different **addresses** or **chains**. One example
could be to listen for `Transfer` events on a couple stablecoins, like USDT, USDC and DAI. Another example could
be cross-chain queries, where you want to run the same query on a different chain with a different smart contract address.
For this, `apollo` has loops. The `loop` block is a top-level element, and always requires an `items` property.
This property will contain the elements you want to loop over. Finally you define your query, with access
to each `item` element. Let's look at an example.

Imagine you want to see the mid prices of both the Sushiswap USDC-ETH pool on Arbitrum, and the Uniswap
USDC-ETH pool on Ethereum. The variables are thus the `chain` and the `address`, which we can store
in the loop `items`. We can then define our query, and the loop elements will be found in `item`.
Make sure to save `item.chain`, otherwise you won't be able to differentiate the output.
```hcl
loop {
  items = [
    {chain = "arbitrum", address = "0x905dfCD5649217c42684f23958568e533C711Aa3"},
    {chain = "ethereum", address = "0xB4e16d0168e52d35CaCD2c6185b44281Ec28C9Dc"},
  ]

  query usdc_eth_reserves {
    // Get the element chain
    chain = item.chain

    contract {
      // Get the element address
      address = item.address
      abi = "unipair.abi.json"

      // Call getReserves
      method getReserves {
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
      // Save chain to differentiate outputs
      chain = item.chain
      usdc_reserve = usdc_reserve
      eth_reserve = eth_reserve
      // Calculate mid price
      mid_price = usdc_reserve / eth_reserve
    }
  }
}
```
## Custom Helper Functions
For some very common operations, `apollo` has helper functions. For now, these are
limited to `balance`, for getting native balances, and `token_balance`, for getting
ERC20 token balances. See some examples [here](./schema-examples.md#query-native-balance)
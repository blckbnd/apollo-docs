# Introduction
The schema is in the form of a DSL implemented with [HCL](https://github.com/hashicorp/hcl) to define the data
we're interested in. This means that basic arithmetic operations and ternary operators
for control flow are supported by default. The top-level elements are `chain` and `contract`.
In the `contract` block we provide the ABI file, along with which `methods` or `events` we want to get the data for.

In the case of a `method`, we first define `inputs` and `outputs`. For an `event`, it's only `outputs`.
The names of the methods, events, inputs and outputs should correspond exactly to what's in the provided
ABI file.

The last block in `contract` is the `save` block. In this block we can do some basic transformations
before saving our output, and it provides access to variables and functions that we might need. 

### Save Context
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

Below are some annotated examples to help you get started. There are some more examples in the [docs](docs/schema/schema-examples.md).
### Methods Example
```hcl
// Define the chain to run on
chain = "arbitrum"

contract usdc_eth_reserves "0x905dfCD5649217c42684f23958568e533C711Aa3" {
  // Will search in the Apollo config directory
  abi = "unipair.abi.json"

  // Call methods
  method getReserves {
    // These are the outputs we're interested in. They are available 
    // to transform as variables in the "save" block below. Outputs should
    // be provided as a list.
    outputs = ["_reserve0", "_reserve1"]
  }

  // The "save" block will give us access to more context, including variables
  // like "timestamp", "blocknumber", "contract_address", and any inputs or outputs
  // defined earlier.
  save {
    timestamp = timestamp
    block = blocknumber
    contract = contract_address
    eth_reserve = parse_decimals(_reserve0, 18)
    usdc_reserve = parse_decimals(_reserve1, 6)

    // Example: we want to calculate the mid price from the 2 reserves.
    // Cannot reuse variables that are defined in the same "save" block.
    // We have to reuse variables that were defined in advance, i.e.
    // in "inputs" or "outputs"
    mid_price = parse_decimals(_reserve1, 6) / parse_decimals(_reserve0, 18)

  }
}
```
### Events Example
```hcl
// Define the chain to run on
chain = "arbitrum"

contract usdc_to_eth_swaps "0x905dfCD5649217c42684f23958568e533C711Aa3" {
  // Will search in the Apollo config directory
  abi = "unipair.abi.json"

  // Listen for events
  event Swap {
    // The outputs we're interested in, same way as with methods.
    outputs = ["amount1In", "amount0Out", "amount0In", "amount1Out"]
  }


  // Besides the normal context, the "save" block for events provides an additional
  // variable "tx_hash". This is the transaction hash of the originating transaction.
  save {
    timestamp = timestamp
    block = blocknumber
    contract = contract_address
    tx_hash = tx_hash

    // Example: we want to calculate the price of the swap.
    price = amount0Out != 0 ? (parse_decimals(amount1In, 6) / parse_decimals(amount0Out, 18)) : (parse_decimals(amount1Out, 6) / parse_decimals(amount0In, 18))
    dir = amount0Out != 0 ? upper("buy") : upper("sell")
    size = amount1In != 0 ? parse_decimals(amount1In, 6) : parse_decimals(amount1Out, 6)
  }
}
```
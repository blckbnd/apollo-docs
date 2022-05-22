---
sidebar_position: 2
---
# Getting Started
## Installation
## Setting up
First, generate the config directory and files:
```
apollo init
```
This will generate the configuration files (`config.yml` and `schema.hcl`) and put it into your configuration
directory, which will either be `$XDG_CONFIG_HOME/apollo` or `$HOME/.config/apollo`. This is the directory
in which you have to configure `apollo`, and it's also the directory where `apollo` will try to find the specified
contract ABIs.

## Running
### Realtime mode
In realtime mode, we only have to define the interval if we're doing a method calling schema (in seconds) and the chain, 
and an optional output option (either `--csv`, `--db` or `--stdout`). See the [Output](##Output) for more info on that.

**Examples**

* Run a method calling schema every 5 seconds in realtime on Arbitrum, and save the results in a csv
```
apollo --realtime --interval 5 --csv
```
* Run an event collecting schema in realtime on Ethereum, save the results in a database
```
apollo --realtime --db
```

### Historical mode
In historical mode, we define the start and end blocks, the chain, the interval (when we're doing method calls),
and an optional output option.

**Examples**

* Run a method calling schema every 100 blocks with a start and end block on Arbitrum, and save the results in a DB and a csv
```
apollo --start-block 1000000 --end-block 1200000 --interval 100 --csv --db
```
* Run an event collecting schema over a range of blocks on Polygon, and output the results to `stdout`
```
apollo --start-block 1000000 --end-block 1500000 --stdout
```

## Output
There are 3 output options:
* `stdout`: this will just print the results to your terminal.
* `csv`: this will save your output into a csv file. The name of your file corresponds to the `name` field of your contract schema definition. The other columns are going to be the inputs and outputs fields, also with the names corresponding to the schema.
* `db`: this will save your output into a Postgres SQL table. The settings are defined in `config.yml` in your `apollo`
config directory.

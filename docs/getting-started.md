---
sidebar_position: 2
---
# Getting started
## Installation
### With Go
```bash
go install github.com/chainbound/apollo
```

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
By default, `apollo` will be running in historical mode, according to the parameters defined in the schema.
To run it in realtime mode, pass the `--realtime` flag.
In realtime mode, we only have to define the interval if we're doing a method calling schema (in seconds) and the chain, 
and an optional output option (either `--csv`, `--db` or `--stdout`). See the [Output](##Output) for more info on that.
* **Example**: Run an event schema in realtime and save the output in a CSV file:
```bash
apollo --realtime --csv
```
* **Example**: Run a method calling schema every 5 seconds, save the result in a database:
```bash
apollo --realtime --interval 5 --db
```
### Historical mode
With historical mode, the only required parameter is an output option. The 
time parameters have to be defined in the schema.

## Output
There are 3 output options:
* `stdout`: this will just print the results to your terminal.
* `csv`: this will save your output into a csv file. The name of your file corresponds to the name of your query. 
* `db`: this will save your output into a Postgres SQL table. The settings are defined in `config.yml` in your `apollo`
config directory.

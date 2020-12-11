# Redis Backed State

The `Default` state implementation holds the data in-memory using a `HashMap`, but in fact anything can be used to store the values. `Redis` is a 
great option to store key/value pairs and to allow different ways of accessing the data with its special data structure commands.

## State
In this case the `State` would store the values in `Redis`, the `Connection` layer would have to implement a reasonable way of exchanging the 
data either by exchanging changes or the whole Redis `RDB` file and the `Agent` can be either agnostic to the fact that the values are 
stored in `Redis` or by exposing `Redis` commands through HTTP to allow an application to make actual calls to `Redis`.

The `State` would store values in `Redis`. Since the `StateValue` can represent anything, the `State` can treat it as a command protocol to pass 
to `Redis`. For example, the `StateValue` can be a `JSON` object that describes a `Redis` command.

## Connection
The `Connection` layer would have to implement a reasonable way of exchanging the `Redis` data with other peers. Either be exchanging changes only, 
or by exchanging the full `Redis RDB` file.

## Agent
The `Agent` can actually be agnostic to the fact that the state is stored in `Redis`. The `Default` agent, for example, does not know anything about 
the payload of the data so the data can be a `JSON` object that represents a command to `Redis` or it can be a normal key value object to be stored 
in `Redis` without being to specific to the fact that it's `Redis`.

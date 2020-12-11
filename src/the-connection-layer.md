# The Connection Layer

The connection layer is the low-level layer that is responsible for exchanging the state with other C19 peers.
This is usually the layer that is doing most of the work. One has to consider different use cases and what should be optimized. For 
example, if the use case is for big data, then maybe it's not a good idea to exchange the full state on every change.

The `Default` connection layer publishes only the changes and does so at a specified interval. It also runs a thread to retrieve the full state 
from other peers, given that it holds a different version of the state than the connected peer. This should balance the size of the data with the 
rate of publishing changes.

```rust, no_run
pub trait Connection: std::fmt::Debug + Send + Sync {
    fn start<'a>(
        &'a self,
        state: state::SafeState,
    ) -> BoxFuture<'a, Result<(), Box<dyn StdError + Send + Sync>>>;
}
```

Similar to the `Agent` trait, the connection trait requires one method: `start` which accepts the `state` instance.

The `Default` connection layer initializes the publisher and receiver threads in this function.

## What to Consider When Implementing a Connection Layer
The connection layer is usually the most involved layer to implement. It must consider how the state should be exchanged. At what rate, which peers to choose, 
what type of protocol to use when exchanging the state, etc... It is better first to decide and focus on the use case you are trying to solve.

### Choosing Peers
The way the connection layer chooses peers to exchange the state with affects how the data is propagated throughout the system.

The connection to other peers can be a persistent connection or a stateless connection. The peers chosen can be at random or using a specific formula.

The `Default` connection layer chooses peers at random, uses HTTP calls to exchange the state with and disconnects. It does so at a specified interval.

### Protocol
When connecting to other peers the state should be exchanged. There are many ways to exchange the data. Depending on the use case you wish to solve, 
you might want to exchange only the changes since last exchange, or maybe to negotiate a version to be exchanged or maybe just the full state.

The `Default` connection layer publishes the changes at a specified interval and pulls the full state at a different interval. When pulling the full state it specifies the 
current version it has and if it matches the one that the peer holds then nothing is exchanged.

### Rate
Should the connection layer exchange the state on every change in state? Or maybe at a certain interval?
If the state changes often and the connection layer exchanges the state on every change then you gain an immediate representation of the state across the system but at 
the expense of resources. Publishing too slow will extend the time the system is unbalanced in terms of the shared state across the system.

The `Default` connection layer publishes the state at a specified, configurable interval. This decoupling of exchanging the state with the change of state allows the state 
to be changed at any rate without affecting the resources for exchanging it.

### Supporting Peer Providers
Peer providers allow a connection layer to query for available peers to choose from. A connection layer implementation might choose a different way to get the list of available 
peers, but it's recommended to support peer providers since it allows for greater extensibility. By decoupling the logic of listing available peers from exchanging data, the user 
will have more options to choose from. 

The `Default` connection layer uses a peer provider to get the list of available peers and then chooses a few at random.

You will learn more about peer providers in a following section.

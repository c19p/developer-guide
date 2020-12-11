# Peer Providers

When a connection layer needs to select peers to connect to it can use a `Peer Provider` to get the full list of available peers to choose from.

The `Default` connection layer is using a peer provider to get the full list of available peers. 

## The StaticPeerProvider

The `StaticPeerProvider` is a simple peer provider that allows to statically (manually) configure the list of available peers. This is useful mainly 
for development and testing your work locally.

```yaml
peer_provider:
  kind: Static
  peers:
    - 192.168.1.2
    - 192.168.1.3:3000
```



The static peer provider allows setting a fixed, predefined list of peers. You can configure it to include as many peers as you wish and can also 
include a different port for each peer. 

It is up to the connection layer to resolve the port to use when connecting to other peers. The `Default` connection layer uses the `target_port` and 
the `port` configuration fields as a fallback if a specific port is not defined in the list of available peers.

The `K8s` peer provider does not even return a port. It only lists the pod IP addresses.

We will look at the `StaticPeerProvider` implementation in detail when we walk through the `Default` layer implementations.

## The PeerProvider Trait
```rust, no_run
pub trait PeerProvider: std::fmt::Debug + Send + Sync {
    /// Initializes the peer provider.
    fn init(&self) -> Result<(), Box<dyn StdError + Send + Sync>>;

    /// Returns a vector of available peers.
    fn get(&self) -> Vec<Peer>;
}
```

The peer provider is first initialized by the framework. It must expose a `get` function that is expected to return the list of available peers the connection 
layer should choose from.

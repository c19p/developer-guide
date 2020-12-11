# The StaticPeerProvider

A peer provider is responsible for returning a list of available peers for the connection layer to choose from.

The static peer provider allows configuring a predefined, static list of peer endpoints.

## The Peer Provider Trait
```rust, no_run
pub trait PeerProvider: std::fmt::Debug + Send + Sync {
    /// Initializes the peer provider.
    fn init(&self) -> Result<(), Box<dyn StdError + Send + Sync>>;

    /// Returns a vector of available peers.
    fn get(&self) -> Vec<Peer>;
}
```

As seen in the architecture chapter, the `PeerProvider` trait requires a peer provider to implement a `get` function that returns a vector of peers.

A peer is defined as follows:
```rust, no_run
pub enum Peer {
    Ipv4Addr(Ipv4Addr),
    SocketAddrV4(SocketAddrV4),
}
```

A peer is an enum with either an Ipv4 or a full socket address. The reason behind this is to allow a peer provider to return a vector of ip address or 
ip:port.

## The Static Peer Provider
```rust, no_run
#[derive(Serialize, Deserialize, Debug)]
pub struct Static {
    peers: Vec<Peer>,
}

#[typetag::serde]
impl PeerProvider for Static {
    fn init(&self) -> Result<()> {
        Ok(())
    }

    fn get(&self) -> Vec<Peer> {
        self.peers.clone()
    }
}
```

The `Static` struct is loaded by the configuration module and is expecting a vector of `ip` or `ip:port` values.
```yaml
peer_provider:
  kind: Static
  peers:
    - 192.168.1.2
    - 192.168.1.3:3000
```

The `get` function implementation just returns that list.


Feel free to browse the `K8s` peer provider code to see a more complex implementation for a peer provider.

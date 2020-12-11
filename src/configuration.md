# Configuration

When the `main` function runs, the first thing it does is to load the configuration.

The configuration is a YAML file with a main part and a section for each of the layers.

```yaml
version: 0.1
spec:
  agent:
    kind: Default
    port: 3097
  state:
    kind: Default
    ttl: null
    purge_interval: 60000
  connection:
    kind: Default
    port: 4097
    push_interval: 1000
    pull_interval: 60000
    r0: 3
    timeout: 1000
    peer_provider:
      kind: K8s
      selector:
        c19: getting-started
      namespace: default
```

The `Config` class dynamically initializes each section using `serde_yaml` and returns a configuration object which holds each section. The `kind` 
field for every section tells `serde` which layer to load. 

When the `run` function of the project is called, all three layers are initialized with this configuration and then started.

When implementing one of the layers, your struct might look like this:

```rust, no_run
/// The default state struct.
#[derive(Serialize, Deserialize, Debug, Clone)]
#[serde(default)]
pub struct Default {
    /// The default TTL (in milliseconds) to use if none is specified when setting a new value.
    ttl: Option<u64>,

    /// The interval in milliseconds in which to purge expired values.
    ///
    /// Default value is 1 minute (60000 milliseconds).
    purge_interval: u64,

    /// The DataSeeder to use for seeding the data on initialization.
    data_seeder: Option<Arc<RwLock<Box<dyn DataSeeder>>>>,

    /// The version of the current state.
    ///
    /// This is set to a random unique string on every state change.
    #[serde(skip_serializing, skip_deserializing)]
    version: Arc<RwLock<String>>,

    /// The SyncSender channel to use for async set operations
    ///
    /// When a set operation is being commited to the state, the state 
    /// will pass the operation to an async handler which will then commit the 
    /// changes to the state.
    #[serde(skip_serializing, skip_deserializing)]
    tx: Option<mpsc::SyncSender<Vec<u8>>>,

    /// The data storage in the form of a Key/Value hashmap.
    #[serde(skip_serializing, skip_deserializing)]
    storage: Arc<RwLock<HashMap<String, Box<Value>>>>,

    /// Calculating the version is a bit expensive so we use 
    /// the dirty flag to lazily calculate the verison on-demand.
    #[serde(skip_serializing, skip_deserializing)]
    is_dirty: Arc<RwLock<bool>>,
}
```

If we compare this to how the `Default` state configuration looks like:
```yaml
  state:
    kind: Default
    ttl: null
    purge_interval: 60000
```

The `ttl`, `purge_interval` and `data_seeder` fields are loaded but all other fields are skipped. This is because only the `ttl`, `purge_interval` and `data_seeder`
should be configurable while the other members of the struct are for internal use. You can mark members to be skipped with `serde`'s skip annotations.

# The Default State

In this section we will walk through the implementation of the `Default` state layer. By the end of this chapter you should be able to implement 
your own state layer.

## Behavior
Before diving into the code, let's first try to describe the state behavior.

### The Store
The `Default` state layer holds a HashMap of `String` keys to `Values`.
A value is a structure that holds an optional TTL, a timestamp and the value itself which is a `serde_json::Value`.

When the state layer receives a `StateValue` to be set, it will deserialize it into a HashMap and will merge it into its own HashMap.

Put in other words, the `Default` state layer expects the value to be a JSON object with an optional TTL, a timestamp and a value which can be 
anything JSON compatible.

### Resolving Conflicts
To resolve conflicts in case two different peers hold different versions of the same key, the state layer uses a timestamp. The timestamp is set 
when a key is being committed to the state. This means that a newer timestamp will always "win" the conflict.

## The Code
Let's explore the code.

### The `Default` struct

This struct holds the configuration of the state and implements the `State` trait.

```rust, no_run
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

`ttl` - The `Default` state layer offers an option to set a default TTL value for keys that do not specify an explicit TTL.

`purge_interval` - The state will never return expired keys but will hold them in memory until purged at a certain interval for the sake of performance.

`data_seeder` - The state supports data seeders and will load the data when it first initializes.

`version` - The state calculates the state version by combining (xor) the hash of all the keys and their respective timestamps. Note that the version is 
behind a lock to conform to the thread-safety requirement.

`tx` - Setting a large value to the state might be CPU intensive operation. The state will do this in the background on a dedicated thread. This field 
is used for writing new `set` operations.

`storage` - The HashMap to hold the data. Note that the HashMap is behind a lock since we have to make sure our state is thread-safe.

`is_dirty` - For performance reasons, the state will use this flag to do some operations in a lazy manner. Calculating the state version, for example, is a costly 
operation. We want to calculate the version only on-demand and when the data has changed.

As you will learn in a following chapter, a few of the fields above are loaded by the configuration component of the C19 agent.
Specifically, the ttl, purge_interval and the data_seeder are configured by the user.

### The Value
The `Value` struct describes the values that we hold in our state.

```rust, no_run
struct Value {
    /// A serde_json::Value to hold any value that can be serialized into JSON format.
    value: serde_json::Value,

    /// The timestamp when this value was first created.
    #[serde(default = "epoch")]
    ts: u64,

    /// An optional TTL (resolved to an absolute epoch time) when this value will be expired.
    ttl: Option<u64>,
}
```

`value` - A `serde_json::Value` to hold the value itself.

`ts` - A timestamp representing the time when the key was first created.

`ttl` - An optional TTL for this value.

### Implementing the `State` Trait
Let's explore a few of the more interesting functions of the `State` trait.

### Init
To conform to the `State` trait we first have to implement the `init` function. 
```rust, no_run
    fn init(&self) -> SafeState;
```

The `init` function is expected to return a `SafeState` which is a type definition of an `Arc`.
```rust, no_run
    fn init(&self) -> state::SafeState {
        let mut this = self.clone();

        // if we have a data seeder then use it to seed the data
        this.data_seeder.clone().and_then(|data_seeder| {
            info!("Seeding data...");
            if let Err(e) =  this.seed(data_seeder) {
                warn!("Failed to seed data; ({})", e);
            }

            Some(())
        });

        // start the async_set consumer thread
        let (tx, rx) = mpsc::sync_channel(MAX_SET_OPS);
        this.tx = Some(tx);

        let this = Arc::new(this);
        let t = this.clone();
        tokio::task::spawn_blocking(move || {async_set(t, rx)});
      
        // start the purger thread
        tokio::spawn(purge(this.clone()));

        this
    }
```

The `Default` state first loads a data seeder, if one is provided, and the goes on to initialize some internal threads. One of async set operations and one for purging expired 
values.

In the end it returns an `Arc::new` that wraps self.

### Get
```rust, no_run
    fn get(&self, key: &dyn StateValue) -> Option<Box<dyn StateValue>> {
        let key: String = String::from_utf8(key.as_bytes().unwrap_or(Vec::new())).unwrap();

        let storage = self.storage.read().unwrap().clone();
        storage
            .get(&key)
            .cloned()
            .filter(|v| !v.is_expired())
            .map(|v| v.into())
    }
```

The state first deserialize the key to a String. It is up to the caller to make sure the key is a `StateValue` that represents a `String`.

Then, the state unlocks the storage for reading (recall that the storage is behind a read lock) looks up the key and filters it out if it has expired.

### Set
```rust, no_run
    fn set(&self, map: &HashMap<String, Box<Value>>) {
        let map = map.clone();
        let mut is_dirty = false;

        // merge the maps
        let mut storage = self.storage.write().unwrap();
        for (key, mut right) in map {
            if right.is_expired() {
                continue;
            }

            if self.ttl.is_some() && right.ttl.is_none() {
                right.ttl = self.ttl;
            }

            storage.entry(key)
                .and_modify(|v| {
                    if v.ts < right.ts {
                        *v = right.clone().into();
                        is_dirty = true;
                    }})
            .or_insert({
                is_dirty = true;
                right.into()
            }); 
        }

        *self.is_dirty.write().unwrap() = is_dirty;
    }

```

The `Default` state layer implements async set operations by running a dedicated thread that consumes a channel. The `set` function above is called by the async set
operation after it has deserialized the `StateValue` into a `HashMap` to make the actual set.

Here, the state merges the maps while marking the state as `dirty`. It does so to allow `lazy` operations to be performed only if the state has changed.

### Implementing the `StateValue`
To implement a `StateValue` we have to implement a deserialization function that will deserialize our value into a vector of `u8`.

#### `StateValue` Trait
```rust, no_run
pub trait StateValue: Send + Sync {
    fn as_bytes(&self) -> Option<Vec<u8>>;
}
```

### `Value` implementing the `StateValue` Trait
```rust, no_run
impl StateValue for Value {
    fn as_bytes(&self) -> Option<Vec<u8>> {
        serde_json::to_vec(self).ok()
    }
}
```

This way the connection and agent layers can pass a `StateValue` around without needing to know anything about it.

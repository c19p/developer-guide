# The State Layer

As mentioned, the state layer is responsible for holding and managing the data. It exposes ways for the Connection and Agent layers 
to set and get values to and from the state.

When implementing a `State` layer, one must conform to the `State` trait.

```rust, no_run
/// The State trait.
///
/// Every state implementor must implement this trait. It is first auto-loaded by the configuration
/// deserializer and then initialized by the library by calling the `init` function. The `init`
/// function should return a [SafeState] which is then passed to the connection and agent layers.
#[typetag::serde(tag = "kind")]
pub trait State: std::fmt::Debug + Send + Sync + CloneState {
    /// Initializes the state and returns a state that is safe to be shared across threads.
    ///
    /// A state object is already loaded by the configuration. The implementor can use this
    /// function to add or initialize any other relevant data and then return a SafeState object
    /// which is shared with the connection and agent layers.
    fn init(&self) -> SafeState;

    /// Returns the version of the current state.
    ///
    /// An implementor can use this function to keep a version for each "state" of the sate. For
    /// example, the default state implementation sets this value to 
    /// a random string whenever the state changes. It is then saves that version in a version history which 
    /// allows for the connection layer to compute diff between two versions. 
    fn version(&self) -> String;

    /// Sets a value to the state.
    ///
    /// There's no assumption about the value itself. It can be anything the implementor wishes.
    /// The default state implementation, for example, treats this value as a map of key/value
    /// pairs where the key is a String and the value conforms to a serde_json::Value value.
    fn set(&self, value: &dyn StateValue) -> Result<(), Box<dyn StdError>>;

    /// Gets the value associated with the specified key.
    ///
    /// To allow maximum flexibility, the key itself is a StateValue, which in effect means it can
    /// be anything desired by the implementor.
    fn get(&self, key: &dyn StateValue) -> Option<Box<dyn StateValue>>;

    /// Returns the value associated with the specified key or the default if the key was not found 
    /// in the state.
    fn get_or(&self, key: &dyn StateValue, default: Box<dyn StateValue>) -> Box<dyn StateValue> {
        self.get(key).unwrap_or(default)
    }

    /// Returns the difference between this and the `other` state.
    fn diff(&self, other: &dyn StateValue) -> Result<Box<dyn StateValue>, Box<dyn StdError>>;

    /// Returns the whole state as a StateValue.
    ///
    /// This is helpful when the connection layer wishes to publish the whole state to its peers.
    fn get_root(&self) -> Option<Box<dyn StateValue>>;
}

```

As you can see from the code snippet of the `State` trait, a few functions must be implemented.

`init` - When the state layer is first being loaded by the framework, the `init` function is called and is expected to return a `SafeState` instance 
of the state. A `SafeState` is just a typedef to the state wrapped in an `Arc` so it can be safely shared between the different layers. It is 
up to the implementation to make sure the state is thread-safe. 

The `Default` state layer implementation uses this function to initialize a few background threads.

`version` - The `version` function can be used by a Connection layer to optimize exchanging states with other peers. The `Default` connection 
layer implementation does not publish a state that has already been published before. It also uses it when retrieving a state from other peers by 
specifying the version it has. The `Default` state layer calculates the version by hashing the keys and timestamps for each key. Each implementation 
should choose what's best for its use case. Imagine a `Git` state layer where the version is the `HEAD`.

`set` - This is the function used by the other layers to set values into the state. You can see that it accepts a `StateValue`. You will learn more 
about the `StateValue` later on in this chapter, but for now it's important to note that the `StateValue` can represent any data structure. The connection 
and agent layers do not know anything about the value. They just pass a vector of bytes to the state and it's up to the state to deserialize it to the proper 
data structure.

The `Default` state assumes the `StateValue` is a JSON object with a key and a value that contains different fields (ttl, ts and the value itself).

`get` - This function is called by the other layers to get a value from the state. Same as the `set` function, it accepts a `StateValue` that should 
represent something that the state knows how to deal with. The `Default` state assumes this is a string that represents the key to be retrieved.

`get_or` - Is a convenience function that allows to specify a default value if the key being retrieved does not exist.

`diff` - This function should calculate the diff between this state and the state that is specified as a parameter. The `Default` connection layer calls 
this function when it wants to publish the changes since last publish time.

`get_root` - This function is called by other layers when they need the whole state. The `Default` connection layer uses it to get the full state to be exchanged 
with other peers.

## What to Consider
When implementing a State layer, we have to consider a few things:

### How do we hold the data?
The data is being serialized and deserialized by the state layer. You must first decide what data structure will be used to represent the data.
The `Default` state layer holds the data as a HashMap that maps a String key to a Value. The Value is in itself a struct that holds information about 
the value and the value itself which is a `serde_json::Value` object.

Or maybe the data should be held in a local DB like Redis or SQLite? You can then have your state get and set values to that DB instead of holding it yourself.

### What do we expect from the Connection and Agent layers when they get and set values?
Since the user choose whatever combination of layers they want, the state layer cannot assume anything about the Agent and Connection layers. On the other 
hand it has to expose ways for the agent and connection layers to get and set values to it. As you will learn in a following section, this is done by a struct that 
implements the `StateValue` trait. A `StateValue` is something that can be serialized and deserialized to a byte vector. 

The agent and connection layers pass values to and from the state by form of a byte vector. They do not assume anything about the meaning of this byte vector. It can 
be a JSON, binary data or any other structure. It is up to the state layer to decide what it should be.

### What other properties of the data we implement?
Except for the value itself that is set by the user, a state layer can decide on other features to be included with a value. The `Default` state layer, for example, allows 
setting a TTL for keys.

### How should conflicts be resolved?
Since the C19 protocol is a distributed system, one must consider a case where a C19 agent is being updated by two other peers in parallel, each one setting the same key. One value might 
be older than the other. It is up to the state layer to resolve the conflict.

Imagine a case where C19 agent `A` is holding a key `k` with a certain value. Another C19 agent `B` is holding the same key but with an updated value that C19 agent A wasn't yet updated to.
If they both update a C19 agent C then agent C would have to resolve the conflict of which value for key `k` it should hold.

The `Default` state layer resolves conflicts by considering a timestamp for each key when it was first created. It will always choose the value that has the most recent timestamp.

### Thread safety
The state is being passed around the layers which operates concurrently using different threads and futures. When implementing a `State` layer, you should make sure your data is 
being protected from concurrent access. Put in other words, your state is a shared resource.

### Performance
The connection and agent layers trust the state layer to be performant when setting and getting values. If the state layer is slow to respond it might affect the agent when replying to 
the application or the connection layer when exchanging data with other peers. You should be mindful of the performance when you make decisions on implementation. The `Default` state layer 
uses async set operations to make sure it doesn't block readers of the state. This way when the connection layer is setting a full state it got from another peer, the `get` operations performed 
by the agent layer will not be blocked.

The `Default` state holds the data in a HashMap which is behind a `RwLock`.

### Supporting Data Seeders
Data seeders are a way to seed data into a C19 agent when it first loads. You will learn more about data seeders in a following chapter.
It is recommended for a state layer to support data seeders.

The `Default` state layer loads a data seeder when it first initializes.

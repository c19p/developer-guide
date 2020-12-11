# The State Value

Before we can continue to talk about the different layers, we have to talk about the `StateValue`.

As mentioned, the `StateValue` is how the different layers talk to each other. If the `Agent`, `Connection` and `State` 
cannot assume anything about one another, how can they share the data between them?

The `StateValue` is a trait that requires a single function to serialize the data into a byte vector.
It is up to the state layer to determine how the data should be treated. The agent and connection layers are merely responsible 
for transferring the data as-is from the consumer to the state. 

```rust, no_run
pub trait StateValue: Send + Sync {
    fn as_bytes(&self) -> Option<Vec<u8>>;
}
```

As you can see, the implementer of a `StateValue` must implement the `as_bytes` function that serializes the value into a `u8` vector.

Here's how the `Default` state layer implements the `StateValue` trait:
```rust, no_run
impl StateValue for Value {
    fn as_bytes(&self) -> Option<Vec<u8>> {
        serde_json::to_vec(self).ok()
    }
}
```

As you will learn when we walk through the `Default` state layer implementation, the `Value` is a struct that holds a single value in the state.
The value is JSON compatible so the `Default` state uses the `serde_json` crate to serialize the value into a `u8` vector.

When deserializing a u8 vector into a `Value` struct, the `Default` state layer assumes the u8 vector represents a valid JSON data. It's up to 
the user to make sure they conform to the state requirements.

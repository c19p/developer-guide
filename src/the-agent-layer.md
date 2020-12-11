# The Agent Layer

The agent layer is the entry point of the application to get and set values to the state. It should expose ways for the application to do that.

The `Default` agent layer exposes two HTTP endpoints for setting and getting values.
We will learn more about the `Default` agent layer when we walk through the `Default` layer implementations.

```rust, no_run
pub trait Agent: std::fmt::Debug + Send + Sync {
    fn start<'a>(
        &'a self,
        state: state::SafeState,
    ) -> BoxFuture<'a, Result<(), Box<dyn StdError + Send + Sync>>>;
}
```
The `Agent` trait requires implementation of a single function: `start` to allow the agent layer to initialize different threads or futures.
The `Default` agent initializes the HTTP server at this point.

As you can see in the `start` function signature, it accepts a `state` instance. This allows the agent layer to communicate with the state.

## What to Consider When Implementing an Agent Layer
The point of interaction for the Agent layer is the state on one side and the application on the other side. There's nothing much expected from 
the agent layer when it comes to setting and getting values to the state, but much is expected when talking to the application.

### Performance
The agent layer should be performant when it responds to the application. It is dependent on the state layer for its performance but can and should 
still do what it can do respond in a timely manner to the application.

### API
There's no hard constraint when it comes to the API that is exposed to the application. It's up to you to decide what you wish to expose to the application. 

The `Default` agent layer exposes HTTP endpoints to get and set values. One might implement a persistent connection (WebSocket for example) or add options for the 
application to be notified immediately when there's a change to the state.

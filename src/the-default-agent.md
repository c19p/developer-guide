# The Default Agent

The `Default` agent layer implementation allows an application to get and set values to the state through HTTP calls.

It exposes two endpoints: `GET` and `PUT` and does not assume anything about the payload itself. It is up to the application (user) 
to make sure the payload conforms to the chosen `State` layer.

Any `Agent` implementation should choose its own way of exposing ways to get and set values to the state.

## Behavior
The behavior of the `Default` agent is straight forward. It runs a HTTP server with two handlers for getting and setting values.
```rust, no_run
async fn handler(state: state::SafeState, req: Request<Body>) -> Result<Response<Body>> {
    Ok(match req.method() {
        &Method::GET => get_handler(state, &req),
        &Method::PUT => set_handler(state, req).await.unwrap(),
        _ => Responses::not_found(None),
    })
}

impl Default {
    async fn server(&self, state: state::SafeState) -> Result<()> {
        let service = make_service_fn(move |_| {
            let state = state.clone();
            async move {
                Ok::<_, Box<dyn StdError + Send + Sync>>(service_fn(move |req| {
                    handler(state.clone(), req)
                }))
            }
        });

        let server = Server::try_bind(&([0, 0, 0, 0], self.port).into())?;
        server.serve(service).await?;

        Ok(())
    }
}

```

## The `Agent` Trait
```rust, no_run
pub trait Agent: std::fmt::Debug + Send + Sync {
    fn start<'a>(
        &'a self,
        state: state::SafeState,
    ) -> BoxFuture<'a, Result<(), Box<dyn StdError + Send + Sync>>>;
}
```

The `Default` agent layer uses the `start` function to initialize and run the HTTP server.
```rust, no_run
fn start<'a>(&'a self, state: state::SafeState) -> BoxFuture<'a, Result<()>> {
    self.server(state).map_err(|e| e.into()).boxed()
}
```

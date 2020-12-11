# The Default Connection

The connection layer is responsible for exchanging the state with other peers. As mentioned before, this is probably the more involved layer to implement. 
It is involved because one has to decide how the state should be exchanged and there are usually trade-offs to make. Exchanging to fast will be resource intensive, 
while exchanging to slow will extend the imbalance of the shared state. 

## Behavior
The `Default` connection layer uses two threads to exchange the state with other peers: The publisher and receiver threads.

The `publisher` thread publishes the only the changes made since the last publish time. 
The `receiver` thread will connect and pull the full state from a peer if they both hold a different version of the state.

Apart from the two threads for exchanging the state, the connection layer runs a HTTP server to allow other peers to connect and exchange the state with it.

It exposes two endpoints: one for retrieving the state if the other for setting the state.

## Get
```rust, no_run
fn get_handler(state: state::SafeState, req: &Request<Body>) -> Response<Body> {
    let versions_match = req.uri().path().split('/').last().and_then(|version| {
        trace!("get_handler / version {} ({})", version, state.version());

        (version.is_empty() || version != state.version()).into()
    }).unwrap();


    if versions_match {
        Responses::ok(state.get_root().unwrap().into())
    } else {
        Responses::no_content()
    }
}
```

Here, the `Default` connection layer compares the version it holds against the version reported by the connected peer. If the versions do not match then 
it will return the full state. Otherwise nothing will be returned.

## Set
```rust, no_run
fn set_handler<'a>(
    state: state::SafeState,
    req: Request<Body>,
) -> impl FutureExt<Output = Result<Response<Body>>> + 'a {
    hyper::body::to_bytes(req.into_body())
        .and_then(move |body| async move {
            let body = &body as &dyn state::StateValue;
            let result = state.set(body);

            Ok(match result {
                Ok(_) => Responses::no_content(),
                _ => Responses::unprocessable(None),
            })
        })
        .map_err(|e| e.into())
}
```

The `Default` connection layer exposes this function to allow other peers to publish their state. It will accept the state and set it to the `State` layer.

## The Receiver Thread
The receiver thread connects to other peers and sends a HTTP `GET` request while specifying the version it holds. 
It uses a peer provider to get the full list of available peers and randomly chooses a few of them.

It them loops through the responses and sets them to the state.

```rust, no_run
    async fn receiver(&self, state: state::SafeState) -> Result<()> {
        loop {
            // sample r0 peers
            let peers = self.peer_provider.get();
            let peers = peers.into_iter().sample(self.r0);
          
            let res = stream::iter(peers)
                .map(|peer| {
                    let url = format!("http://{}:{}/{}", peer.ip(), peer.port().unwrap_or(self.target_port.unwrap_or(self.port)), state.version());
                    let timeout = self.timeout;

                    tokio::spawn(async move {
                        trace!("Retreiving from {}", url);
                        let client = Client::builder()
                            .connect_timeout(Duration::from_millis(timeout))
                            .build().unwrap();

                        let result = client
                            .get(&url.to_string())
                            .send()
                            .await?
                            .bytes()
                            .await;

                        result
                    })
                })
                .buffer_unordered(4);

            res.collect::<Vec<_>>().await.iter().for_each(|result| {
                if let Ok(Ok(result)) = result {
                    if let Err(e) = state.set(result as &dyn state::StateValue) {
                        warn!("Failed to set peer response to state; {}", e);
                    }
                }
            });

            time::delay_for(time::Duration::from_millis(self.pull_interval)).await;
        }
    }

```

## The Publisher Thread
The publisher thread connects to other peers and publishes the changes since last publish time. 

```rust, no_run
    async fn publisher(&self, state: state::SafeState) -> Result<()> {
        let mut last_published: Vec<u8> = Vec::<u8>::default();
        let mut last_published_version = String::default();

        loop {
            if last_published_version == state.version() {
                time::delay_for(time::Duration::from_millis(self.push_interval)).await;
                continue;
            }

            let last = last_published.clone();
            let state_clone = state.clone();
            let res = tokio::task::spawn_blocking(move || {
                // get the recent state
                let root = state_clone.get_root().unwrap().as_bytes().unwrap();

                if root == last {
                    return None;
                }

                let state_to_publish = state_clone.diff(&last).and_then(|diff| {
                    Ok(diff.as_bytes().unwrap())
                }).or::<Vec<u8>>(Ok(root.clone())).unwrap();

                Some((state_to_publish, root))
            }).await?;

            if res.is_none() {
                time::delay_for(time::Duration::from_millis(self.push_interval)).await;
                continue;
            }

            let (state_to_publish, last) = res.unwrap();
            last_published = last;
            last_published_version = state.version();

            // sample r0 peers
            let peers = self.peer_provider.get();
            let peers = peers.into_iter().sample(self.r0);

            // start sending to peers in parallel
            let res = stream::iter(peers)
                .map(|peer| {
                    let state_to_publish = state_to_publish.clone();
                    let url = format!("http://{}:{}/", peer.ip(), peer.port().unwrap_or(self.target_port.unwrap_or(self.port)));
                    let timeout = self.timeout;

                    tokio::spawn(async move {
                        trace!("Sending to {}", url);
                        let client = Client::builder()
                            .connect_timeout(Duration::from_millis(timeout))
                            .build().unwrap();

                        let result = client
                            .put(&url.to_string())
                            .body(state_to_publish)
                            .send()
                            .await?
                            .bytes()
                            .await;

                        result
                    })
                })
                .buffer_unordered(4);

            res.collect::<Vec<_>>().await.iter().for_each(|result| {
                if let Ok(Ok(result)) = result {
                    if let Err(e) = state.set(result as &dyn state::StateValue) {
                        warn!("Failed to set peer response to state; {}", e);
                    }
                }
            });

            time::delay_for(time::Duration::from_millis(self.push_interval)).await;
        }
    }
```

[FIXME: no need for the publish state to set the responses to the state since all responses are expected to be no-content]

# Data Seeders

When a C19 agent first launches and joins a C19 cluster, it is expected from the connection layer to connect and exchange the state with other, already running agents. 
This allows a new C19 agent to sync with other peers as it launches.

Nonetheless there are cases where a user would want to initialize the agent with some data. For that we have data seeders.

Depending on the `State` layer implementation, when it first loads it is expected to initialize and load a data seeder. If you choose to implement a `State` layer, 
please consider supporting the `DataSeeder` API so users will have a consistent option to seed your state with data.

## The `File` Data Seeder
The `File` data seeder will load data from a file. The data in the file must be compatible with the `State` layer and it's up to the user to make sure it does.

The `Default` state layer expects the following format:
```json
{
  "cat": {
    "value": "garfield",
    "ttl": 3600000
  },
  "dog": {
    "value": "snoopy"
  }
}
```

When the `Default` state layer is being initialized it loads the data seeder, if one is configured:
```rust, no_run
fn init(&self) -> state::SafeState {
    ...

    // if we have a data seeder then use it to seed the data
    this.data_seeder.clone().and_then(|data_seeder| {
        if let Err(e) =  this.seed(data_seeder) {
            warn!("Failed to seed data; ({})", e);
        }

        Some(())
    });
    
    ...
}
```

## The `DataSeeder` Trait
```rust, no_run
pub trait DataSeeder: std::fmt::Debug + Send + Sync {
    fn load(&self) -> Result<Box<dyn StateValue>, Box<dyn StdError>>;
}

```

A very simple trait with a `load` function that returns a `StateValue`.

We will explore the implementation of the `File` data seeder in detail when we walk through the `Defaul` layer implementations. 

# The File DataSeeder

A data seeder is responsible for seeding data into a state once loaded.

The `File` data seeder loads the content of a file and set it to the state. It is agnostic to the format of the file. It is up to the user 
to make sure the format of the file complies to the `State` requirements.

The `Default` state, for example, requires the data to be of a certain `JSON` format:
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

## The DataSeeder Trait
```rust, no_run
pub trait DataSeeder: std::fmt::Debug + Send + Sync {
    fn load(&self) -> Result<Box<dyn StateValue>, Box<dyn StdError>>;
}

```
We are required to implement the `load` function and return anything that implements the `StateValue` trait. If you recall, the `StateValue` trait 
requires us to implement a function that deserializes our value into a `u8` vector.

```rust, no_run
pub trait StateValue: Send + Sync {
    fn as_bytes(&self) -> Option<Vec<u8>>;
}
```

## The File DataSeeder
```rust, no_run
#[derive(Serialize, Deserialize, Debug)]
pub struct File {
    filename: String,
}

#[typetag::serde]
impl data_seeder::DataSeeder for File {
    fn load(&self) -> Result<Box<dyn StateValue>> {
        let data: Box<dyn StateValue> = Box::new(fs::read(&self.filename)?);
        Ok(data)
    }
}
```

The `File` data seeder expects a `filename` to be specified in its configuration. When the `load` function is called it will read the file and return the data, 
which complies with the `StateValue` trait.

---
sidebar_position: 3
---

# State and Instantiate

Within this section of the tutorial we will be defining the state of our contract, state is
everything that will be stored on chain in the contract's storage. After this is setup we will
define our contract's instantiate entry point which performs the initial setup of our contract
(similarly) to a constructor in object-oriented programming.

## State

All state definitions for our contract will be located within the `src/state.rs` file, let's open
this file and see what boilerplate state has already been generated for us. Upon opening the file
you should see:

```rust
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

use cosmwasm_std::Addr;
use cw_storage_plus::Item;

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct State {
    pub count: i32,
    pub owner: Addr,
}

pub const STATE: Item<State> = Item::new("state");
```

Before we make any changes, let's describe what each component of this file does:

- `JsonSchema` allows a struct to be used when generating JSON Schema, this is useful when
  outputting types for a frontend using `examples/schema.rs`
- `Serialize` and `Deserialize` allow the struct to be serialized and deserialized to and from
  base64 (the `Binary` Cosmwasm type)
- `Clone` allows us to call `.clone()` on instances of the struct to perform a deepcopy
- `Debug` allows us to debug print the struct to the console easily
- `PartialEq` allows us to compare equality of the struct using partial equivalence relations
- `Addr` is a bech32 encoded Cosmos address, although under the hood it performs no validation and is
  simply a `String`
- `Item` is a helper provided by `cw-storage-plus` it represents a single item in storage. In the code above it can store a struct of type `State` under the `storage_key` state

### Config

Now we can make some modifications to this code, firstly let's rename the `State` struct to `Config` as this item will store our overall contract configuration. We also need to remove the `count` and `owner` variables within the struct as we will be adding our own. The file should now appear as follows:

```rust
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

use cosmwasm_std::Addr;
use cw_storage_plus::Item;

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct Config {

}

pub const CONFIG: Item<Config> = Item::new("config");
```

Now we need to fill out our `Config` struct, let's define an admin of type `Addr`. In future
this contract could be expanded to allow the admin to delete polls. Our `Config` struct should
now look like this:

```rust
// Previous code omitted
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct Config {
    pub admin: Addr
}
// Following code omitted
```

With the global configuration complete, let's define the state for our polls.

### Polls and Ballots

Before we start writing code, let's think about what a poll is built up of. We can simply describe a
poll as a question asked by a specific person, with a set list of responses each with a corresponding vote count. Let's see how we can translate this to Rust code.

- Poll creator
  - We know user's will need to submit a TX with a given wallet to create a poll on chain
  - This means we can use the user's `Addr` which is associated with the create poll message sent
- Poll question
  - A question is simply a string of characters
  - We can simply use the Rust `String` type to store this
- Options
  - In other languages we may use a `Map` however a `HashMap` in Rust cannot be saved to chain due to
    consensus issues
  - This mean we need to store our options in a list of tuples
    - These tuples will be in the format of `(String, u64)` where the `String` is the textual name of
      the option and the `u64` is the number of votes for that option
  - In Rust we can represent this as a `Vec<(String, u64)>` which is simply a list of these tuples

Now that we have a specification for a singular poll, let's create the struct for it. Ensure to
include all the `derive` macros `Config` also has defined.

```rust
// Previous code omitted
// Derive the necessary features
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct Poll {
    pub creator: Addr,
    pub question: String,
    pub options: Vec<(String, u64)>,
}
// Following code omitted
```

With this poll defined it exposes a hole in our model, as we simply store an option and its vote count
as a tuple within the `Poll` how do we determine if a user has already voted (so we can
decrement their old vote prior to incrementing their new vote). To do this we need to store a users
vote, let's call it a ballot. All this ballot needs to store is the option they vote for.

```rust
// Previous code omitted
// Derive the necessary features
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct Ballot {
    pub option: String,
}
// Following code omitted
```

As explained above an `Item` stores a singular item in storage, however due to our defined model we
can have multiple polls and a user can have multiple ballots. To store these we will use another
`cw-storage-plus` helper, `Map`. A map simply maps keys of a given type, to a value of another type.

Firstly we need to import it, we can do this by modifying the `Item` import at the top of the file to
also import `Map`.

```rust
// Currently the import should look like this
use cw_storage_plus::Item;
// Update it to look like this
// This now imports both Item and Map
use cw_storage_plus::{Item, Map};
```

Now we need to define a `Map`, to do this we need to give it the type of our key and the type of our
value, as well as a `storage_key` similarly to `Item`. This is how this looks for our polls.

```rust
// Previous code omitted
pub const CONFIG: Item<Config> = Item::new("config");

// A map with a String key and Poll value.
// The key must be unique, this could be a UUID or a generated slug
pub const POLLS: Map<String, Poll> = Map::new("polls");
```

We also need to store the user's ballots, the key for this `Map` will be more complex as one user can
have many ballots. To achieve this using `Map` we can use a tuple key, this key will be of the type
`(Addr, String)` where `Addr` is the voter's address and `String` is the polls ID. This `Map` will of
course have the value type of `Ballot`.

```rust
// Previous code omitted
pub const CONFIG: Item<Config> = Item::new("config");

// A map with a String key and Poll value
// The key must be unique, this could be a UUID or a generated slug
pub const POLLS: Map<String, Poll> = Map::new("polls");

// A map with a tuple key (Addr, String) and a Ballot value
// The tuple is made up of the voter's address and the polls ID
// Example:
// A key of ("wasm1xxx", "1") will point to the vote of address
// wasm1xxx for poll 1
pub const BALLOTS: Map<(Addr, String), Ballot> = Map::new("ballots");
```

We have successfully outlined the state for our contract, however we have broken some of the build
commands (`cargo test`, `cargo wasm`, and `cargo schema`). In the next section we will focus on fixing
these before moving on.

## Fixing the Errors

Due to use renaming the `State` struct this broke our schema output as the non-existent `State` struct
is still imported in `examples/schema.rs`. To fix this we need to replace the import and update the
schema export.

```rust
// The import should look like this
use cw_starter::state::State;

// Modify it to import Config instead
use cw_starter::state::Config;
```

Further down the file there will be a usage of the `State` struct which is also now incorrect, let's
replace this with our `Config` struct.

```rust
// The export_schema line should look like this
// Previous code omitted
export_schema(&schema_for!(State), &out_dir);
// Following code omitted

// Modify it to export Config instead
// Previous code omitted
export_schema(&schema_for!(Config), &out_dir);
// Following code omitted
```

We also defined two new structs (`Poll` and `Ballot`) we should export the schema of these as well
to allow frontends to generate types for our structs. To do this we need to import them and then
export the schema.

```rust
// The import should look like this
use cw_starter::state::Config;

// Modify it to import Config, Poll, Ballot instead
use cw_starter::state::{Config, Poll, Ballot};
```

Now we can export the schema.

```rust
// Previous code omitted
export_schema(&schema_for!(Config), &out_dir);
export_schema(&schema_for!(Poll), &out_dir);
export_schema(&schema_for!(Ballot), &out_dir);
// Following code omitted
```

This has now fixed the errors, run the following commands to verify this:

```bash
# this will generate new JSON files, for our config, poll and ballot
cargo schema
# still 0 tests but should pass
cargo test
# will generate the wasm under target/
cargo wasm
```

## Instantiate

In this sub-section we will implement everything needed to successfully instantiate our contract,
this includes defining the `InstantiateMsg`, writing the instantiate entrypoint and, writing unit
tests for the instantiate entrypoint.

### InstantiateMsg

In order to instantiate the contract we must define the `InstantiateMsg` used, this is how
we can pass values to the contract during it's creation. In our case the only configuration
value is the `admin` stored in the `Config` of the contract.

All messages for our contract are defined in `src/msg.rs`, this section focuses on `InstantiateMsg`
and later chapters will focus on the others. Currently the generated `InstantiateMsg` should look like:

```rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub struct InstantiateMsg {
    pub val: String,
}
```

As you can see it also features the derivations for our state, it also has a serde flag to rename
all the variables within it to be snake case.

We want the user to optionally define an admin for the contract, and if one is not defined it will
default to their address. We can do this in Rust by using the `Option` struct, this effectively allows
a value to be `None` (`null` in other languages) or another given type. In our case we will use
`Option<String>` which means it can be `None` or a `String` value. We are not using the `Addr` type
due to it not being validated, so we will accept any `String` and validate it on the contract side.

```rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub struct InstantiateMsg {
    pub admin: Option<String>, // If admin is None, default to the sender's address
}
```

Now that the message is defined we can begin implementing our instantiate logic.

### Instantiate Entry Point

The instantiate entrypoint is defined in `src/contract.rs` so open that file. Towards the top of the
file we should see the defined entrypoint, currently it should simply contain an `unimplemented!()`
macro that will panic when anything hits this endpoint.

```rust
// Previous code omitted
/*
const CONTRACT_NAME: &str = "crates.io:cw-starter";
const CONTRACT_VERSION: &str = env!("CARGO_PKG_VERSION");
 */

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn instantiate(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    unimplemented!()
}
// Following code omitted
```

To start our implementation we're going to implement the `cw2` specification which allows contracts
to store a name and a version (the commented out code above). Firstly we need to uncomment those.

```rust
// Previous code omitted

const CONTRACT_NAME: &str = "crates.io:cw-starter";
const CONTRACT_VERSION: &str = env!("CARGO_PKG_VERSION");

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn instantiate(
    _deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    unimplemented!()
}
// Following code omitted
```

Next we need to store these values, `cw2` comes with a helper called `set_contract_version` let's import it for use:

```rust
// Previous code omitted
use cosmwasm_std::{Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult};
use cw2::set_contract_version;
// Following code omitted
```

Then we can add it at the top of our instantiate function body.

```rust
// Previous code omitted
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn instantiate(
    deps: DepsMut, // ensure you remove the preceeding _. _deps -> deps
    _env: Env,
    _info: MessageInfo,
    _msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)?;
    unimplemented!()
}
// Following code omitted
```

As we have now modified our function's arguments let's give a run down of what purpose they serve:

- `deps` - The dependencies, this contains your contract storage, the ability to query other
  contracts and balances, and some API functionality
- `env` - The environment, this contains contract information such as its address, block information
  such as current height and time, as well as some optional transaction info
- `info` - Message metadata, contains the sender of the message (`Addr`) and the funds sent with it (`Vec<Coin>`)
- `msg` - The `InstantiateMsg` defined in `src/msg.rs`

So we have stored some contract metadata but have not implemented any poll specific logic. To do this
we need to create our `Config` struct and store it in the `CONFIG` `Item`, firstly let's import these
needed values.

```rust
use crate::state::{Config, CONFIG};
```

Now we have these values we need to determine who the admin is going to be, validate their address,
create the struct, store the struct, and return an `Ok` response.

```rust
// Previous code omitted
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo, // removed _
    msg: InstantiateMsg, // removed _
) -> Result<Response, ContractError> {
    set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)?;
    let admin = msg.admin.unwrap_or(info.sender.to_string()); // if None, use info.sender
    let validated_admin = deps.api.addr_validate(&admin)?; // validate the address
    let config = Config {
        admin: validated_admin.clone(),
    };
    CONFIG.save(deps.storage, &config)?;
    Ok(Response::new()
        .add_attribute("action", "instantiate")
        .add_attribute("admin", validated_admin.to_string()))
}
// Following code omitted
```

The first two lines after setting the contract version, determine what address should be used to
set the admin and then validate the address. The validatie function takes an `&str` so we must prefix
our admin with `&`. This validation will throw an invalid address error on failure meaning we unwrap
the result using `?` to assert success (on failure the code will terminate with this error).

After the validation is performed we create a `Config` struct with the validated admin, this struct
is then saved in the `CONFIG` item using `deps.storage` which is our contracts storage.

The final line returns the response of our instantiate entrypoint with some added metadata, the two
attributes simply tell the user what action they performed and who the admin is. These attributes can
be set to any string value, however it is common to return useful metadata according to your route
(similarly to HTTP headers). This is wrapped in `Ok` to create a `Result` and signal the call was
successful, if we wanted to error we would wrap our error in `Err`.

With this code the instantiate entrypoint is complete, however to demonstrate it working we will

### Instantiate Tests

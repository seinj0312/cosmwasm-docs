---
sidebar_position: 2
---

# Creating the Project

In this section we will be creating the initial project directory and explaining the project
structured generated for us by `cw-template`.

## Using cw-template

To bootstrap the project and generate all the boilerplate and starter code needed we will be
using [cw-template](https://github.com/InterWasm/cw-template). In order to use `cw-template`
Cargo generate must be installed, this can be done by running the following terminal commands
listed in the `README.md`:

```bash
cargo install cargo-generate --features vendored-openssl
cargo install cargo-run-script
```

Once the installation of the dependencies is complete using the above commands we can generate our
project. `cw-template` has two main entry points, using the latest version of the template, or
using an older or alternative branch of the template. The commands for both of these are as follows:

```bash
# using the latest
cargo generate --git https://github.com/CosmWasm/cw-template.git --name <PROJECT_NAME>

# using a specific branch or version
cargo generate --git https://github.com/CosmWasm/cw-template.git --branch <BRANCH_NAME> --name <PROJECT_NAME>
```

For compatibility we will be using a different branch and therefore different version, this means
anyone following along will have the exact same development environment. The branch I will be using
is `1.0-minimal`, this simply bootstraps the project with minimal boilerplate code. This means we
must run the following command to setup our project:

```bash
# using branch 1.0-minimal and naming the project cw-starter
cargo generate --git https://github.com/CosmWasm/cw-template.git --branch 1.0-minimal --name cw-starter
```

The command should output lines generating all the required files and once completed print "Done!"
to the terminal. To verify the project is setup correctly let's run some commands:

```bash
# change directory into the root of the contract
cd cw-starter
# run all tests in the project, currently there are 0
cargo test
# generate JSON schema, generates schema files under the schema directory
cargo schema
# build an unoptimised WASM file, will be located under
# target/wasm32-unknown-unknown/release/cw_starter.wasm
cargo wasm
```

All of these commands should run successfully indicating that the project is setup.

## Project Structure

Let's open the project in our editor and expand all the sub-directories located within it.
There are a lot of files located within it, in this sub-section we will be breaking down
the important ones and what purpose they serve.

```
cw-starter/ # Root
├── .cargo/
│   └── config # Configuration for cargo commands such as cargo wasm, cargo schema, etc.
├── .circleci/
│   └── config.yml # Circle CI workflows and integration
├── .github/workflows/
│   └── Basic.yml # GitHub actions integration
├── artifacts/ # Where optimised WASM files are outputted
├── examples/
│   └── schema.rs # Rust file to generate JSON schema via cargo schema. Outputs to schema/
├── schema/ # Output folder for JSON schema
├── src/ # Where our smart contract rust files are located
│   ├── contract.rs # Main contract logic, instantiate, execute, query
│   ├── error.rs # Where we define our contract errors
│   ├── helpers.rs # Where helper functions for our contract are located
│   ├── lib.rs # Where we define our modules
│   ├── msg.rs # Where we define our message types
│   └── state.rs # Where we define any state variables
├── target/ # Where unoptimised WASM files are outputted
├── Cargo.toml # Define project dependencies
└── Cargo.lock # Lockfile for dependencies
```

The template also comes with other files such as `README.md`, `NOTICE.md` and gitpod integration.
These files will be outside of the scope of this tutorial and are often contract specific e.g. a
contract's `README.md` may differ based on its purpose.

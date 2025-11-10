+++
title = "Config Validator"
summary = "A simple proc-macro to allow the integration of the config and clap crates."
date = 2025-11-06

[taxonomies]
tags = ["rust"]
+++

# Config Validator

This post is about validating user settings during the initialization of your application.
The context here is to allow the user to have multiple ways of configuring the application, so it facilitates different deployment methods and debugging issues.

## Goal
1. The user must be able to configure the application through...
    * Command line arguments 
    * Config file (yaml/json)
    * Environment variables
1. The settings must be easily usable throughout the application.
1. The user must be able to issue different commands to the application in addition to the configuration.


## Solution

There are crates available that can achieve most of the goals above, but I couldn't find one that can do it all.
My solution is to create a proc-macro that allows other crates to be combined so all the goals above can be achieved.

For this solution we will be ...
1. Using the Config crate to load configuration from different sources and apply them in priority order.
1. Using the Clap crate to process command line arguments and build a help command.
1. Implementing the config::Source trait for clap:ArgMatches, so we can use Clap as a Config source.
1. Implementing a proc-macro to validate the overall configuration while allowing optional or mandatory field notation to be used. 

### The Config crate

The Config crate allows combining configurations from different source based on priority. It also provides standard implementations for .yaml and .json files as well as environment variables.
The Config crate can do much more but for the scope of this article this is what you need to know.

Here is a simple example of how to configure the Config crate to parse the settings from a file and then overwrite it with settings from environment variables.

```rust
// filename: config_simple/src/lib.rs
use config::{Config, Environment, File};
use serde::{Deserialize, Serialize};
use std::env;

const DEFAULT_CONFIG_FILE_PATH: &str = "config.yaml";

/// sample-app configuration structure
#[derive(Serialize, Deserialize, Debug, Clone, Default)]
pub struct Settings {
    // optional field
    pub name: Option<String>,
    // optional field
    pub address: Option<String>,
    // mandatory field
    pub phone_number: String,
}


fn run_configuration() -> Settings {
    // app configuration
    let c = Config::builder()
        .add_source(File::with_name(DEFAULT_CONFIG_FILE_PATH).required(false))
        .add_source(Environment::default())
        .build()
        .unwrap();

    c.try_deserialize().unwrap()
}
```

```yaml
# filename: config_simple/config.yaml
name: "Joe"
address: "123 Main St"
phone_number: "555-12
```

### The Clap crate

The Clap crate allows the handling of command line arguments as well as generating a help command to list all options available.
Just like the Config crate, Clap has many other features that I won't be using in this article.

Below is a simple example on how to use the clap crate.
Note that the use of the clap_derive::Parse proc-macro was intentionally avoided, so the application have access to the arguments and commands from the command line.

Check the test section to understand how cli commands can be used with this approach.

```rust
// filename: clap_simple/src/lib.rs
use clap::{ArgMatches, Args, FromArgMatches, Subcommand};
use clap_derive;
use strum_macros::EnumString;

/// configuration structure
#[derive(Debug, Clone, clap_derive::Args)]
pub struct Settings {
    // optional field
    #[arg(short, long)]
    pub name: Option<String>,
    // optional field
    #[arg(short, long)]
    pub address: Option<String>,
    // mandatory field
    #[arg(short, long)]
    pub phone_number: String,
}

/// List available cli commands
#[derive(Debug, clap_derive::Subcommand, PartialEq, EnumString)]
pub enum CliCommands {
    // allow commands to be sent to the application
    #[strum(serialize = "start")]
    #[command(about = "Starts the application")]
    Start,
}

fn run_configuration(arg_vec: Vec<&str>) -> (Settings, ArgMatches) {
    let cli = clap::Command::new("clap-simple");
    let cli = Settings::augment_args(cli);
    let cli = CliCommands::augment_subcommands(cli);
    let mut matches = cli.get_matches_from(arg_vec);

    // Convert ArgMatches -> GreetArgs (FromArgMatches is derived)
    let settings = Settings::from_arg_matches_mut(&mut matches)
        .expect("error parsing arguments into Settings");

    (settings, matches)
}

#[cfg(test)]
mod tests {
    use std::str::FromStr;

    use super::*;

    #[test]
    fn default_config_test() -> () {
        let arg_vec = vec![
            "clap-simple",
            "-n",
            "Jack",
            "--phone-number",
            "555-1234",
            "start",
        ];
        let (settings, matches) = run_configuration(arg_vec);

        assert!(settings.name == Some(String::from("Jack")));
        assert!(settings.address.is_none());
        assert!(settings.phone_number == "555-1234");

        let (command, _args) = matches.subcommand().unwrap();
        let cli_command = CliCommands::from_str(command).unwrap();

        assert!(matches!(cli_command, CliCommands::Start));
    }

    #[test]
    fn help_menu() -> () {
        let arg_vec = vec!["clap-simple", "-h"];
        let _ = run_configuration(arg_vec);
    }
}
```

### Turning Clap into a Config source

Now we are starting to build our solution. The idea is to implement the config::Source trait for clap::ArgMatches.
After that we can add clap as the third config source in the config builder.

Check the test section for usage examples.

```rust
// filename: clap_config_source/src/lib.rs
use clap::ArgMatches;
use config::{Source, Value, ValueKind};
use std::collections::HashMap;

/// A wrapper for `clap::ArgMatches`
#[derive(Debug, Clone)]
pub struct ClapSource {
    matches: ArgMatches,
}

impl ClapSource {
    /// Wraps `ArgMatches` as a ClapSource object, allowing the usage
    /// of this as a `Config` source later on
    pub fn new(matches: &ArgMatches) -> Self {
        ClapSource { matches: matches.to_owned() }
    }
}

/// Implements the `config::Source` trait for ClapSource
impl Source for ClapSource {
    fn clone_into_box(&self) -> Box<dyn Source + Send + Sync> {
        Box::new((*self).clone())
    }

    fn collect(
        &self,
    ) -> std::result::Result<
        HashMap<std::string::String, config::Value>,
        config::ConfigError,
    > {
        let mut config_map = HashMap::new();
        let origin: String = "cli".into();

        for id in self.matches.ids() {
            if let Some(value_kind) = self.extract_value(id.as_str()) {
                config_map.insert(
                    id.to_string(),
                    Value::new(Some(&origin), value_kind),
                );
            }
        }

        Ok(config_map)
    }
}

impl ClapSource {
    fn extract_value(&self, id_str: &str) -> Option<ValueKind> {
        // Try each type in order, returning early on success
        if let Ok(Some(val)) = self.matches.try_get_one::<bool>(id_str) {
            return Some(ValueKind::Boolean(*val));
        }

        if let Ok(Some(val)) = self.matches.try_get_one::<i64>(id_str) {
            return Some(ValueKind::I64(*val));
        }

        if let Ok(Some(val)) = self.matches.try_get_one::<f64>(id_str) {
            return Some(ValueKind::Float(*val));
        }

        if let Ok(Some(val)) = self.matches.try_get_one::<String>(id_str) {
            return Some(ValueKind::String(val.clone()));
        }

        None
    }
}

#[cfg(test)]
pub mod tests {
    use super::*;
    use clap::Args;
    use config::Config;
    use serde::Deserialize;

    /// configuration structure
    #[derive(Deserialize, Debug, Clone, clap_derive::Args)]
    pub struct Settings {
        // optional field
        #[arg(short, long)]
        pub name: Option<String>,
        // optional field
        #[arg(short, long)]
        pub address: Option<String>,
        // mandatory field
        #[arg(short, long)]
        pub phone_number: String,
    }

    #[test]
    pub fn clap_config_source() {
        let cli = clap::Command::new("test");
        let cli = Settings::augment_args(cli);

        let matches = cli
            .try_get_matches_from([
                "test",
                "--name=Joe",
                "--phone-number=555-1234",
            ])
            .unwrap();

        let clap_source = ClapSource::new(&matches);

        let c = Config::builder().add_source(clap_source).build().unwrap();

        let s: Settings = c.try_deserialize().unwrap();

        assert_eq!(s.name, Some(String::from("Joe")));
        assert!(s.address.is_none());
        assert_eq!(s.phone_number, "555-1234");
    }

    #[test]
    #[should_panic()]
    pub fn clap_mandatory_arg() {
        let cli = clap::Command::new("test");
        let cli = Settings::augment_args(cli);

        let matches =
            cli.try_get_matches_from(["test", "--name=Joe"]).unwrap();

    }
}
```

### The Validated proc-macro

Thus far, the solution seems to be working fine, but there are some issues with it.
The main problem is how to deal with optional vs mandatory fields in two different levels, clap and config.

If we define a field in the Settings struct as mandatory (not an Option<_>), the clap source will require the argument in the command line even though the argument may be present in the config file or environment variable.

To illustrate the problem, let's imagine the following example.

A Settings struct with optional and mandatory fields.

```rust
/// configuration structure
#[derive(Deserialize, Debug, Clone, clap_derive::Args)]
pub struct Settings {
    // optional field
    #[arg(short, long)]
    pub name: Option<String>,
    // optional field
    #[arg(short, long)]
    pub address: Option<String>,
    // mandatory field
    #[arg(short, long)]
    pub phone_number: String,
}
```

And let's say we declare the config sources like this:

```rust
let matches = cli
    .try_get_matches_from(["test", "--name=Joe"])
    .unwrap();

// app configuration
let clap_source = ClapSource::new(&matches);
let c = Config::builder()
    .add_source(File::with_name(&DEFAULT_CONFIG).required(false))
    .add_source(File::with_name(&OVERRIDE_CONFIG).required(false))
    .add_source(Environment::default())
    .add_source(clap_source)
    .set_default("phone_number", String::from("555-1234"))?
    .build()
    .unwrap();
```

Since we are parsing the cli first, to pass the arg_matches to the ClapSource object, it will fail because the --phone-number argument is mandatory and was not provided.
The simple solution to this problem is to make all fields in the Settings structure optional.

```rust
/// configuration structure
#[derive(Deserialize, Debug, Clone, clap_derive::Args)]
pub struct Settings {
    // optional field
    #[arg(short, long)]
    pub name: Option<String>,
    // optional field
    #[arg(short, long)]
    pub address: Option<String>,
    // mandatory field
    #[arg(short, long)]
    pub phone_number: Option<String>,
}
```

Now the clap parsing won't fail anymore.

The problem with this approach is that it makes the use of the Settings object inconvenient.
Even though it is safe to use unwrap() on settings.phone_number throughout the application since it has a default value associated with, it is not ideal.

To fix this problem, I created a proc-macro to validate the overall settings and provide concrete access methods to all fields.

Here is an example of how it can be used.

```rust
// filename: config_validator/tests/test.rs
use config_validator::Validate;
use serde::{Deserialize, Serialize};
use std::default;

/// configuration structure
#[derive(
    Serialize, Deserialize, Default, Debug, Clone, clap_derive::Args, Validate,
)]
pub struct Settings {
    // optional field
    #[arg(short, long)]
    pub name: Option<String>,
    // optional field
    #[arg(short, long)]
    pub address: Option<String>,
    // mandatory field
    #[arg(short, long)]
    #[validate(mandatory)]
    pub phone_number: Option<String>,
}

#[test]
#[should_panic(
    expected = r#"[config-validator] mandatory field "phone_number" not provided for struct "Settings""#
)]
fn uninitialized_field_test() {
    let s = Settings::default();
    let _ = s.validate_configuration();
}

#[test]
fn field_getter_test() -> anyhow::Result<()> {
    let s = Settings {
        phone_number: Some(String::from("555-1234")),
        ..Default::default()
    };
    let s: ValidatedSettings = s.validate_configuration();
    let _: String = s.get_phone_number();

    Ok(())
}
```

The trick behind the Validate proc-macro is that it will generate a new struct, in case all mandatory arguments are present, with getters according to each field configuration.
The new struct name is prefixed with "Validated". So in the example above, the settings are present in a struct named ValidatedSettings.

Here is the Validate macro expansion for the example above.
Note that optional field getters return an Option<_> type while mandatory field getters return the concrete type.

```rust
// ======================================
// Recursive expansion of Validate macro
// ======================================

impl Settings {
    pub fn validate_configuration(&self) -> ValidatedSettings {
        if self.config_file.is_none() {
            panic!(
                "[config-validator] mandatory field \"{}\" not provided for struct \"{}\"",
                stringify!(config_file),
                stringify!(Settings)
            );
        }
        if self.log_config_file.is_none() {
            panic!(
                "[config-validator] mandatory field \"{}\" not provided for struct \"{}\"",
                stringify!(log_config_file),
                stringify!(Settings)
            );
        }
        if self.phone_number.is_none() {
            panic!(
                "[config-validator] mandatory field \"{}\" not provided for struct \"{}\"",
                stringify!(phone_number),
                stringify!(Settings)
            );
        }
        ValidatedSettings(self.clone())
    }
}
#[derive(Clone, Debug, Serialize)]
pub struct ValidatedSettings(Settings);

impl ValidatedSettings {
    pub fn get_config_file(&self) -> String {
        self.0.config_file.clone().unwrap()
    }
    pub fn get_log_config_file(&self) -> String {
        self.0.log_config_file.clone().unwrap()
    }
    pub fn get_name(&self) -> Option<String> {
        self.0.name.clone()
    }
    pub fn get_address(&self) -> Option<String> {
        self.0.address.clone()
    }
    pub fn get_phone_number(&self) -> String {
        self.0.phone_number.clone().unwrap()
    }
}
```


### Wrapping all up into a single example

```rust
// filename: config_validator_example/src/settings.rs
use anyhow::Result;
use clap::{ArgMatches, Args, Subcommand};
use clap_config_source::ClapSource;
use config::{Config, Environment, File};
use config_validator::Validate;
use serde::{Deserialize, Serialize};
use strum_macros::EnumString;

const DEFAULT_LOG_CONFIG: &str = "files/log_config.yaml";
const DEFAULT_APP_CONFIG: &str = "files/app_config.yaml";

#[derive(clap_derive::Subcommand, Debug)]
enum Commands {}

#[derive(
    Serialize, Deserialize, Debug, Clone, 
    Default, clap_derive::Args, Validate
)]
pub struct Settings {
    #[arg(short, long, default_value = &DEFAULT_APP_CONFIG)]
    #[validate(mandatory)]
    pub config_file: Option<String>,
    #[arg(short, long)]
    #[validate(mandatory)]
    pub log_config_file: Option<String>,
    #[arg(short, long)]
    pub name: Option<String>,
    #[arg(short, long)]
    pub address: Option<String>,
    #[arg(short, long)]
    #[validate(mandatory)]
    pub phone_number: Option<String>,
}

/// List available cli commands
#[derive(Debug, clap_derive::Subcommand, PartialEq, EnumString)]
pub enum CliCommands {
    #[strum(serialize = "start")]
    #[command(about = "Starts the application")]
    Start,
    #[strum(serialize = "say-hello")]
    #[command(about = "Say Hello back to user")]
    SayHello,
}

impl Settings {
    /// Instantiate a Settings object
    /// * Parse the passed configuration file
    /// * Read environment variables
    /// * Read cli
    ///
    /// ### Returns:
    /// A Validated instance of this Settings and ArgMatches
    pub fn process() -> Result<(ValidatedSettings, ArgMatches)> {
        // cli input
        let cli = clap::Command::new("my-app");
        let cli = Settings::augment_args(cli);
        let cli = CliCommands::augment_subcommands(cli);
        let matches = cli.get_matches();

        // app configuration
        let clap_source = ClapSource::new(&matches);
        let c = Config::builder()
            .add_source(File::with_name(&DEFAULT_APP_CONFIG).required(false))
            .add_source(Environment::default())
            .add_source(clap_source)
            .set_default("log_config_file", DEFAULT_LOG_CONFIG)?
            .build()
            .unwrap();

        let s: Settings = c.try_deserialize().unwrap();
        let s: ValidatedSettings = s.validate_configuration();

        Ok((s, matches))
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn config_test() -> Result<()> {
        let c = Config::builder()
            .add_source(File::with_name("cfg/config.yaml").required(false))
            .set_default("log_config_file", DEFAULT_LOG_CONFIG)?
            .build()
            .unwrap();

        let _: Settings = c.try_deserialize()?;

        Ok(())
    }
}
```

```rust
// filename: config_validator_example/src/main.rs
mod settings;

use std::str::FromStr;
use log::info;
use anyhow::{Result, anyhow};
use settings::Settings;
use crate::settings::{CliCommands, ValidatedSettings};

fn my_function(settings: ValidatedSettings) {
    info!("config_file: {:?}", settings.get_config_file());
    info!("log_config_file: {:?}", settings.get_log_config_file());
    info!("address: {:?}", settings.get_address());
    info!("name: {:?}", settings.get_name());
    info!("phone_number: {:?}", settings.get_phone_number());
}

fn main() -> Result<()> {
    // load settings
    let (settings, matches) = Settings::process()?;

    // load log configuration
    log4rs::init_file(
        settings.get_log_config_file(), 
        Default::default()).map_err(|e| anyhow!("[log_config] {e}")
    )?;

    let Some((command, _)) = matches.subcommand() else {
        return Err(anyhow!("no command provided"));
    };

    let cli_command = CliCommands::from_str(command)?;

    match cli_command {
        CliCommands::Start => { 
            info!("command start");
            my_function(settings);
        },
        CliCommands::SayHello => info!("Hello User"),
    }
    
    Ok(())
}
```

```yaml
# filename: config_validator_example/files/app_config.yaml
name: "Jack"
phone_number: "555-4321"
```

```yaml
# filename: config_validator_example/files/log_config.yaml
appenders:
  stdout:
    kind: console
    encoder:
      pattern: " {{{l}}} {m}{n}"

root:
  level: info
  appenders:
    - stdout
```

Here are some sample outputs from the example above.

```
config_validator_example % cargo run -- -h

List available cli commands

Usage: config-validator-example [OPTIONS] [COMMAND]

Commands:
  start      Starts the application
  say-hello  Say Hello back to user
  help       Print this message or the help of the given subcommand(s)

Options:
  -c, --config-file <CONFIG_FILE>          [default: files/app_config.yaml]
  -l, --log-config-file <LOG_CONFIG_FILE>  
  -n, --name <NAME>                        
  -a, --address <ADDRESS>                  
  -p, --phone-number <PHONE_NUMBER>        
  -h, --help                               Print help
```

```
config_validator_example % cargo run -- start

 {INFO} command start
 {INFO} config_file: "files/app_config.yaml"
 {INFO} log_config_file: "files/log_config.yaml"
 {INFO} address: None
 {INFO} name: Some("Jack")
 {INFO} phone_number: "555-4321"
```

```
config_validator_example % cargo run -- --phone-number "555-1234" start 

 {INFO} command start
 {INFO} config_file: "files/app_config.yaml"
 {INFO} log_config_file: "files/log_config.yaml"
 {INFO} address: None
 {INFO} name: Some("Jack")
 {INFO} phone_number: "555-1234"
```

```
config_validator_example % PHONE_NUMBER="555-7890" cargo run -- start

 {INFO} command start
 {INFO} config_file: "files/app_config.yaml"
 {INFO} log_config_file: "files/log_config.yaml"
 {INFO} address: None
 {INFO} name: Some("Jack")
 {INFO} phone_number: "555-7890
```

```
config_validator_example % cargo run -- SayHello                        

error: unrecognized subcommand 'SayHello'

  tip: a similar subcommand exists: 'say-hello'

Usage: config-validator-example [OPTIONS] [COMMAND]

For more information, try '--help'.
```

```
config_validator_example % cargo run -- say-hello

 {INFO} Hello User
```

For these final examples the phone_number argument has been removed from the app_config.yaml file.

```yaml
# filename: config_validator_example/files/app_config.yaml
name: "Jack"
```

```
config_validator_example % cargo run -- start                           

thread 'main' panicked at config_validator_example/src/settings.rs:17:76:
[config-validator] mandatory field "phone_number" not provided for struct "Settings"
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

```
config_validator_example % PHONE_NUMBER="555-7890" cargo run -- start

 {INFO} command start
 {INFO} config_file: "files/app_config.yaml"
 {INFO} log_config_file: "files/log_config.yaml"
 {INFO} address: None
 {INFO} name: Some("Jack")
 {INFO} phone_number: "555-7890"
 ```

 ```
config_validator_example % cargo run -- --phone-number "555-1234" start 

 {INFO} command start
 {INFO} config_file: "files/app_config.yaml"
 {INFO} log_config_file: "files/log_config.yaml"
 {INFO} address: None
 {INFO} name: Some("Jack")
 {INFO} phone_number: "555-1234"
 ```

 The code presented in this article is available on [GitHub](https://github.com/vicentecarvalho/blog-projects/tree/main/posts/config_validator).
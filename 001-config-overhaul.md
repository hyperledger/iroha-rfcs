# Iroha 2 Configuration Overhaul RFC

## Introduction

Configuration is often the first point of interaction for users with Hyperledger Iroha 2. It's crucial to make this
experience as smooth as possible. However, the current system, marred by issues like redundant fields, ambiguous naming,
and unclear error messages, necessitates a thorough review. This RFC sets out to propose vital improvements for a more
intuitive and efficient configuration experience.

## Background

### Redundant Fields

There are configuration fields that shouldn't be initialized by the user, meaning that including them in the
configuration reference is redundant. For example, here's the
[excerpt](https://github.com/hyperledger/iroha/blob/35ba182e1d1b6b594712cf63bf448a2edefcf2cd/docs/source/references/config.md#sumeragi-default-null-values)
about configuration options for Sumeragi:

> **Sumeragi: default `null` values**
>
> A special note about sumeragi fields with `null` as default: only the `trusted_peers` field out of the three can be
> initialized via a provided file or an environment variable.
>
> The other two fields, namely `key_pair` and `peer_id`, go through a process of finalization where their values are
> derived from the corresponding ones in the uppermost Iroha config (using its `public_key` and `private_key` fields) or
> the Torii config (via its `p2p_addr`). This ensures that these linked fields stay in sync, and prevents the programmer
> error when different values are provided to these field pairs. Providing either `sumeragi.key_pair` or
> `sumeragi.peer_id` by hand will result in an error, as it should never be done directly.

As we can see, if the user tries to initialize `key_pair` and `peer_id` via the config file or env variables, they'll
get an error. We are creating a terrible user experience by presenting the user with an option to configure something
they should not be configuring.

We should not expose such fields to the user as they are implementation details.

### Naming Issues

The configuration parameter naming convention poses the following challenges.

#### Overcomplicated ENV Variables

ENV variables for all configuration parameters are named according to the internal module names. These technical
designations might resonate with developers but can be confusing or alienating for users not deeply immersed in the
project's inner mechanisms. The
[POSIX specification](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap08.html#tag_08) actively
encourages environment variables with succinct names. Given that a number of standard environment variable names exist,
and their value can be redefined on a per-process basis, it is recommended to use generic environment variables as much
as possible, before using specific names prefixed with `IROHA_`.

#### Inconsistent Naming of ENV Variables

The output from the `iroha --help` command displays a mix of naming conventions:

```
Iroha 2 is configured via environment variables:
    IROHA2_CONFIG_PATH is the location of your `config.json` or `config.json5`
    IROHA2_GENESIS_PATH is the location of your `genesis.json` or `genesis.json5`
...
    IROHA_TORII: Torii (gateway) endpoint configuration
    IROHA_SUMERAGI: Sumeragi (emperor) consensus configuration
    IROHA_KURA: Kura (storage). Configuration of block storage
    IROHA_BLOCK_SYNC: Block synchronisation configuration
...
```

The inconsistency between the `IROHA2` and `IROHA` prefixes can be confusing. Of the two, the `IROHA` prefix is
preferred, as users are discouraged from using Iroha 1 in any new projects.

#### Inconsistent Naming of Configuration Parameters

Current configuration demonstrates some inconsistencies in parameter naming, leading to potential user confusion. To
address this, some refinements are proposed:

- The `torii` section uses terms like `p2p_addr`, `api_url`, and `telemetry_url`. These are actually addresses in the
  form of `localhost:8080` and not full URLs, which would include components like protocol and path. To standardize,
  these should all be changed to use `addr` (e.g., `addr_p2p`, `addr_api`, `addr_telemetry`).
- Consider moving addresses to their respective domain scopes. For example, instead of `torii.telemetry_url`, we could
  have `telemetry.addr`.
- In the `logger` section, the term `max_log_level` could be simplified to `log_level` or just `level` to make it more
  concise without losing meaning. Although users are more likely to set this parameter using an environment variable.
- In the `wsv` section, the field `wasm_runtime_config` has a redundant `_config` suffix. Given these are all
  _configuration_ parameters, such suffixes can be eliminated for clarity.

Implementing a consistent naming convention like this will simplify the user's configuration process and reduce
ambiguity.

#### Redundant SCREAMING_CASE in JSON Configuration

The adoption of SCREAMING_CASE for parameters within the JSON configuration appears out of place. Such uppercase field
names might be customary for environment variables, but in a JSON setting, they don't offer any distinctive benefit.
Here's a snippet from the current configuration showcasing this:

```json
{
  "PUBLIC_KEY": null,
  "PRIVATE_KEY": null,
  "DISABLE_PANIC_TERMINAL_COLORS": false
}
```

---

In summary, these issues underscore the need for clearer, more intuitive naming conventions in Iroha 2's configuration.

### Rustisms in the Configuration Reference

Configuration reference contains Rust-specific expressions that wouldn't make sense to the end-user who is configuring
Iroha and might not be familiar with Rust. Here are some examples:

- [Using `Option<..>`](https://github.com/hyperledger/iroha/blob/35ba182e1d1b6b594712cf63bf448a2edefcf2cd/docs/source/references/config.md#option)
- [Using `std::path::PathBuf`](https://github.com/hyperledger/iroha/blob/35ba182e1d1b6b594712cf63bf448a2edefcf2cd/docs/source/references/config.md#loggerlog_file_path)

The use of rustisms like that overcomplicates the configuration reference and affects the user experience. Further, it
doesn't actually provide any extra information to a user familiar with Rust, as environment variables and JSON values
cannot be `None` due to being stringly typed and null-able regardless of whether the type specifies `Option`.

### Unhelpful Error Messages

Some of the error messages related to Iroha configuration are inconsistent and misleading. Here's a quick rundown of
various scenarios, associated errors and possible confusions.

#### 1. No configuration file

Running Iroha without a configuration file results in the following log and error:

```
IROHA_TORII: environment variable not found
IROHA_SUMERAGI: environment variable not found
IROHA_KURA: environment variable not found
IROHA_BLOCK_SYNC: environment variable not found
IROHA_PUBLIC_KEY: environment variable not found
IROHA_PRIVATE_KEY: environment variable not found
IROHA_GENESIS: environment variable not found
Configuration file not found. Using environment variables as fallback.
Error:
   0: Please add `PUBLIC_KEY and PRIVATE_KEY` to the configuration.
```

Several things are wrong with this error message:

- The information about using environment variables as fallback comes after the messages about them not being found.
- Hinting to add `PUBLIC_KEY` and `PRIVATE_KEY` to the configuration does not explain what happened (the absence of the
  config file and no environment variables to fall back to). This hint is also not helpful as there is no information on
  whether public and private keys should be added to the config file or to ENV variables, and no information about the
  ENV variables these fields are mapped to.

#### 2. Path to non-existent config file

Providing a path to a configuration file that doesn't exist results in the exact same error as if there was no
configuration file specified at all.

When a user specifies a path to a configuration file, it is generally with the expectation that the file will be read
and its settings applied. If the file doesn't exist, an immediate and clear error message should be thrown to alert the
user of this issue.

#### 3. Empty config file

Running Iroha with an empty `config.json` results in the error that does not provide any information about the fallback
to the environment variables even though it does happen. This behaviour is inconsistent.

```
Error:
   0: Please add `PUBLIC_KEY and PRIVATE_KEY` to the configuration.
```

#### 4. Config file with only `PUBLIC_KEY` and `PRIVATE_KEY` specified

Running Iroha with a config file that only contains `PUBLIC_KEY` and `PRIVATE_KEY` results in the following error:

```
Error:
   0: Please add `p2p_addr` to the configuration.
```

This error message is misleading and inconsistent:

- If the `p2p_addr` field is added to the config file, the error stays the same.
- The `p2p_addr` field is not a root-level field, it's actually a part of the `TORII` configuration and should be
  `TORII.P2P_ADDR`.
- Error messages asking to add public and private keys use uppercase for configuration fields, while in this error the
  field name is written in lowercase.

#### 5. Config file with only `PUBLIC_KEY`, `PRIVATE_KEY`, and `TORII.P2P_ADDR` specified

Running Iroha with a config file that only contains `PUBLIC_KEY`, `PRIVATE_KEY`, and `TORII.P2P_ADDR` results in the
following error:

```
Error:
   0: Please add `api_url` to the configuration.
```

Comparing this scenario to the previous one, we can notice that the `TORII.API_URL` was missing before as well but the
previous error message didn't mention it.

There are also two inconsistencies:

- Similar to `p2p_addr` in the previous scenario, this field name is lowercase, while for public and private keys it was
  uppercase.
- While `TORII.P2P_ADDR` and `TORII.API_URL` share the same root-level (`TORII`), their names are different: "addr"
  derived from "address" and "URL". Logically these are both addresses or URLs, and it would make sense for the names to
  be aligned.

#### 6. Config file with invalid `PUBLIC_KEY`

Providing an invalid `PUBLIC_KEY` in the `config.json` results in the same error as when there is no config file at all,
or an empty config file:

```
Error:
   0: Please add `PUBLIC_KEY and PRIVATE_KEY` to the configuration.
```

This is not a helpful error message as it does not mention what exactly is wrong, why and where the parsing failed.

#### 7. Invalid `IROHA2_PUBLIC_KEY` environment variable

Providing an invalid `IROHA2_PUBLIC_KEY` environment variable results in the following error:

```
Error:
   0: Failed to build configuration from env
   1: Failed to deserialize the field `IROHA_PUBLIC_KEY`: JSON5: Key could not be parsed. Key could not be parsed. Invalid character 's' at position 4
   2: JSON5: Key could not be parsed. Key could not be parsed. Invalid character 's' at position 4
```

While this message is more helpful than the ones we discussed above, there are still issues:

- The message contains repetition.
- Without an input snippet, the `Invalid character 's' at position 4` part of the message is not as helpful as it could
  be.

#### 8. Config file with extra fields

Providing a configuration file with extra fields that are not supposed to be there does not produce any errors at all.

This might lead to a bad user experience when user expects some options to apply, but doesn't have any idea that those
in fact are silently ignored.

#### 9. Specifying `SUMERAGI.PUBLIC_KEY`

As you can see in the "Redundant fields" section, user should not specify any non-null value for `SUMERAGI.PUBLIC_KEY`
parameter. However, if they do, they will get the following error:

```
Error:
   0: Please add `PUBLIC_KEY and PRIVATE_KEY` to the configuration.
```

The problem is that the error message tells something completely unrelated to the actual cause. This produces a terrible
experience for users.

#### 10. Config file is an invalid JSON file

Providing an invalid JSON file as a config results in the same error we've seen before, which is not useful at all in
this particular case and doesn't tell the user anything about the invalid JSON:

```
Error:
   0: Please add `PUBLIC_KEY and PRIVATE_KEY` to the configuration.
```

### Ambiguity in Setting Root-Level Fields via ENV

To understand the issue with ambiguity at the root-level fields, let's once again look at the excerpt from the
[configuration reference](https://github.com/hyperledger/iroha/blob/35ba182e1d1b6b594712cf63bf448a2edefcf2cd/docs/source/references/config.md#network):

> **`network`**
>
> Network configuration
>
> Has type `Option<network::ConfigurationProxy>`. Can be configured via environment variable `IROHA_NETWORK`
>
> ```json
> {
>   "ACTOR_CHANNEL_CAPACITY": 100
> }
> ```

The current configuration treats namespaces like `network` as both a grouping mechanism for specific fields (e.g.,
`network.*`) and as independent parameters. This creates ambiguity: for instance, what would happen if an environment
variable like `IROHA_NETWORK` is set? Would it affect nested fields such as `IROHA_NETWORK_CAPACITY`?

This dual role for namespaces is not just limited to `network`, but applies to all other namespaces in the
configuration. This approach is somewhat exotic and not well-documented, leading to potential confusion and
unpredictability. Given these challenges, it might be beneficial to simplify the configuration by treating namespaces
strictly as prefixes for other parameters, rather than as standalone configuration values.

### Chaotic Code Organisation

Internally, not all the configuration-related logic is contained within a single `iroha_config` crate. Instead, some of
the configuration resolution logic is located in other crates, such as `iroha_cli`. This makes the configuration-related
code error-prone and harder to maintain.

### Configuration Endpoints

Iroha's API currently provides two endpoints related to configuration:

- `POST /configuration`: Allows runtime updating of the configuration. As of now, it only supports modifying the log
  level using a request body structured as `{"LogLevel":"<level>"}`.
- `GET /configuration`: Serves a dual purpose; it can either return the entire configuration in a JSON format or provide
  documentation for a specific field.

The inclusion of this information aims to provide context for subsequent proposals discussed in this document.

### Configuration ISIs

Iroha 2's blockchain state, also known as the World State View, is manipulated by transactions containing Iroha Special
Instructions (ISI). Among these instructions, there are specific ones designed to create (`NewParameter`) and modify
(`SetParameter`) **chain-wide** configuration parameters. As of now, these parameters encompass `BlockTime`
(`sumeragi.block_time`), `CommitTimeLimit` (`sumeragi.commit_time_limit`), and `MaxTransactionsInBlock`
(`sumeragi.max_transactions_in_block`).

The ISI set related to configurations isn't extensively documented, and there are concerns about its design and future
applicability. While this RFC does not aim to redefine Iroha 2's instruction set, it's important to consider the
existence of these configuration-related instructions for the context of subsequent sections.

### Relative Paths Resolution

Several configuration parameters in Iroha accept file system paths, such as `kura.block_store_path`. These parameters
can also use relative paths, like the default value `./storage`. However, there's an inconsistency in how these relative
paths are currently resolved.

At present, paths are determined based on the current working directory (CWD) – the directory from which Iroha was
initiated. For instance, if a configuration file with the `./storage` value is located in `/foo/bar/config.json`, but
Iroha starts from `/baz`, the storage will materialise at `/baz/storage` instead of the expected `/foo/bar/storage`.

While this behaviour isn't harmful, resolving paths relative to the configuration file's location might be more
intuitive and expected for users.

### Configuration Similarities with Iroha Client CLI

Iroha 2 comes bundled with an official Rust-based client known as `iroha_client_cli`. The configuration of this client
shares many similarities with Iroha's configuration since both are built upon the same foundational principles. While
the primary focus of this RFC is to address configuration challenges in Iroha, the guidelines and solutions proposed
here are equally applicable to `iroha_client_cli`. However, any changes to the client's configuration would be of
secondary priority.

## Proposals

The proposals outlined in this section are designed to collectively enhance the configuration system within Iroha.
However, they are not mutually required. Each proposal can be considered and implemented independently.

### Proposal 1 - Use TOML

#### Objective

Transition to TOML as Iroha's standard configuration format to provide a cleaner, more intuitive setup and to align with
Rust community standards.

#### Rationale

- **Human-Friendly Syntax:** TOML's format is easier to read and understand compared to JSON, especially given its
  support for comments.
- **Rust Ecosystem Alignment:** Adopting TOML ensures that Iroha remains consistent with prevalent configuration
  practices in the Rust community.
- **Elimination of SCREAMING_CASE:** By using TOML, Iroha can also abandon the use of SCREAMING_CASE in configuration,
  making it more in line with conventional formatting standards.

#### Summary

Adopting TOML allows Iroha to embrace a more readable and conventional configuration format. The move away from JSON
will simplify user interaction and align Iroha better with the broader Rust ecosystem.

#### See Also

- [Proposal 3 - Reconsider Parameter Naming Conventions](#proposal-3---reconsider-parameter-naming-conventions)
- [Proposal 4 - Better Aliases for ENV](#proposal-4---better-aliases-for-env)
- [Proposal 6 - Define Deprecation and Migration Policy](#proposal-6---define-deprecation-and-migration-policy)

### Proposal 2 - Reference Before Implementation

#### Objective

Establish a clear and comprehensive configuration reference prior to actual code implementation. This approach ensures
that any new features or changes are well-documented, understandable, and in alignment with the project's goals.

#### Rationale

- **Clarity and Direction:** By laying out a detailed reference before diving into coding, we make sure everyone's on
  the same page and knows the direction we're headed.
- **Efficient Development:** Prevents potential backtracking or revisions in the coding phase, as developers will be
  working with a clear guide.
- **Enhanced Collaboration:** A preliminary reference can be critiqued, discussed, and iterated upon, leading to better
  decisions in the design phase.
- **User Engagement:** Early availability of a configuration reference aids in early user feedback, ensuring that the
  implementation is user-centric.

#### Sub-points and References:

- **[Proposal 1 - Use TOML](#proposal-1---use-toml)**: Settling on TOML as the standard configuration format before
  diving into the coding phase will prevent format-switching complications.
- **[Proposal 3 - Reconsider Parameter Naming Conventions](#proposal-3---reconsider-parameter-naming-conventions)**:
  Prior to implementation, deciding on naming conventions and potential changes (like nesting keys under `iroha` or
  changing parameter names) will streamline development and ensure consistency.
- **[Proposal 4 - Better Aliases for ENV](#proposal-4---better-aliases-for-env)**: Defining environment variable aliases
  in advance allows for more predictable and user-friendly configuration handling.
- **[Proposal 6 - Define Deprecation and Migration Policy](#proposal-6---define-deprecation-and-migration-policy)**:
  Establishing guidelines on how to handle deprecated features or migrations ensures smoother transitions and fewer
  surprises for users.

#### Summary

Developing a configuration reference before the actual implementation can lead to a more coherent, user-friendly, and
efficient development process. By addressing key points beforehand, like naming conventions, format choices, and alias
definitions, Iroha 2 will be better positioned to deliver a configuration system that resonates with its users.

#### See Also

- [[suggestion] Remove the requirement to set `sumeragi.key_pair` and `sumeragi.peer_id` to `null` in the configuration · Issue #3504 · hyperledger/iroha](https://github.com/hyperledger/iroha/issues/3504)
- [[suggestion] Get rid of "rustisms" in the configuration reference · Issue #3505 · hyperledger/iroha](https://github.com/hyperledger/iroha/issues/3505)
- [[suggestion] Enhance configuration reference format · Issue #3507 · hyperledger/iroha](https://github.com/hyperledger/iroha/issues/3507)

### Proposal 3 - Reconsider Parameter Naming Conventions

#### Objective

The objective of this proposal is to establish a consistent and intuitive naming convention for configuration
parameters, as well as to separate 'developer-only' fields from regular ones.

#### Rationale

1. **Unified Naming Scheme**: Adopting `snake_case` uniformly across all configuration parameters will bring about a
   more straightforward and less error-prone user experience.
2. **Developer-Only Fields**: The introduction of nested `[<module>.dev]` sections will isolate parameters primarily
   used for development purposes, reducing the likelihood of unintentional settings adjustments by end-users.
3. **Specific Renamings**: Some existing field names are either misleading or not fully representative of their
   functionality. By renaming these, we enhance clarity and maintainability.

#### Developer-Only Sections

Introduce `[<module>.dev]` sections to isolate 'developer-only' fields. These won't be included in the user
configuration reference but will be documented separately.

```
# Old
[sumeragi]
debug_force_soft_fork = true

# New
[sumeragi.dev]
force_soft_fork = true
```

#### Specific Renamings

This section details specific field renamings aimed at enhancing clarity, accuracy, and consistency. The changes are as
follows:

- **Socket Addresses**: Replace the `url` keyword with `address` for fields that represent socket addresses (e.g.,
  `localhost:8080`) to avoid confusion. Therefore:
  - `torii.api_url` becomes `torii.api_address`
  - `torii.telemetry_url` becomes `torii.telemetry_address`
- **Key Configuration**: Simplify `iroha.public_key` and `iroha.private_key` to `public_key` and `private_key`
  respectively, eliminating redundant namespace information. (footnote: [[80ad19 Footnote - Keys Configuration]])
- **Telemetry Split**: Separate telemetry configuration into distinct namespaces for better isolation and clarity:
  - `telemetry.substrate`: This will include `name`, `url`, `min_retry_period`, and `min_retry_delay_exponent`.
  - `telemetry.file-output`: This will only include the `file` field.
- **Network Configuration**: Move `sumeragi.p2p_addr` to `network.address` for more contextual naming.
- **Logger Configuration**:
  - Change `max_log_level` to either `log_level` or simply `level` to clarify its purpose.

**Implementation Note**: This RFC may not be updated to reflect all new parameters or renamings that may appear during
the implementation process.

#### Summary

This proposal unifies naming to `snake_case`, separates dev-only fields, and improves field clarity. Further changes may
not be reflected in this RFC.

#### See Also

- [Proposal 1 - Use TOML](#proposal-1---use-toml)
- [Proposal 4 - Better Aliases for ENV](#proposal-4---better-aliases-for-env)
- [Proposal 6 - Define Deprecation and Migration Policy](#proposal-6---define-deprecation-and-migration-policy)
- [Representation of key pair in configuration files · Issue #2135 · hyperledger/iroha](https://github.com/hyperledger/iroha/issues/2135)

### Proposal 4 - Better Aliases for ENV

#### Objective

Introduce intuitive and standardized environment variable names, moving away from internal code-names, to simplify user
experience.

#### Rationale

- **User Accessibility:** Generifying names improves understanding and avoids confusion. Using internal module names can
  be restrictive and lead to misconfiguration.
- **Consistency:** A uniform naming convention for environment variables enhances predictability, aiding users in
  setting up configurations without constantly referring to documentation.
- **Transition Support:** New aliases will coexist with existing ones, ensuring no disruptions and aiding users during
  the transitional phase.

#### Proposed Changes

1. **Rename Based on Functionality:** Shift from code-based naming to function-based naming. For example:
   - `TORII_API_URL` becomes `API_ENDPOINT_URL`.
   - Convert `KURA_INIT_MODE` to `BLOCK_REVALIDATION_MODE`.
   - `LOGGER_MAX_LOG_LEVEL` becomes `MAX_LOG_LEVEL` or simply `LOG_LEVEL`
2. **Document All Aliases:** Clearly list main variables and their respective aliases in the configuration reference.
3. **Trace Resolution in Logs:** For clarity, ensure that when an alias is used, its resolution to the actual parameter
   is evident in logs.

#### Summary

By transitioning to more user-friendly ENV variable names and reducing reliance on internal naming conventions, we aim
to make Iroha 2's configuration process more intuitive and error-free.

#### See Also

- [Proposal 2 - Reference Before Implementation](#proposal-2---reference-before-implementation)
- [Proposal 3 - Reconsider Parameter Naming Conventions](#proposal-3---reconsider-parameter-naming-conventions)
- [Proposal 8 - Trace Configuration Resolution](#proposal-8---trace-configuration-resolution)

### Proposal 5 - Human-Readable Types

#### Objective

The objective of this proposal is to enhance the user experience by allowing the specification of certain configuration
parameters using human-readable types, specifically for durations and byte sizes.

#### Rationale

1. **Enhanced Readability**: Using human-readable strings such as "1m 5s" for durations and "5MB 4KiB" for byte sizes
   enhances the readability of configuration files, making them more accessible for non-technical users.
2. **Reduced Error-Prone Nature**: Human-readable types are less prone to mistakes compared to using raw numerical
   values, which often require users to manually calculate conversions (e.g., minutes to milliseconds, megabytes to
   bytes).

#### Proposed Changes

1. **Durations**: Allow duration fields to be specified in a more human-friendly format. Rather than requiring
   milliseconds as an integer (e.g., `305000` for 305 seconds), support string inputs like `"5minutes 5seconds"`.
2. **Byte Sizes**: Similarly, for fields representing data sizes, support string inputs like `"4KiB"` or `"5MB"`, as an
   alternative to just raw byte integers.

```toml
# Old
cache_size = 5242880
timeout = 1500

# New
cache_size = "5MB"
timeout = "1s 500ms"
```

#### Summary

This proposal introduces human-readable types for duration and byte-size fields, making configuration more intuitive and
less error-prone. The changes will require updates to the parsing logic of the configuration system.

### Proposal 6 - Define Deprecation and Migration Policy

#### Objective

To establish a clear and structured policy for marking features as deprecated and guiding users towards adopting newer
alternatives or configurations.

#### Rationale

- **User Trust:** A well-documented deprecation policy ensures that users can trust the software's lifecycle and
  understand the trajectory of its development.
- **Smooth Transitions:** Clear policies guide users in migrating from older configurations or features to newer ones,
  ensuring continuity of operations.
- **Maintainability:** Developers benefit from clear guidelines on when and how to retire older code, helping in
  reducing technical debt.
- **Clear Communication:** A standardized policy ensures that all users, from developers to end-users, have a clear
  understanding of changes.

#### Proposed Policy

1. **Notification:** When a feature or configuration is marked for deprecation, it should be communicated via release
   notes, documentation updates, and, when relevant, through warnings within the software itself.
2. **Grace Period:** Provide users a sufficient grace period (e.g., a few versions ahead) before the deprecated feature
   is removed. This period allows them to adjust and migrate without sudden disruptions.
3. **Migration Guidelines:** Offer detailed documentation on how to transition from the deprecated feature or
   configuration to its modern replacement.
4. **Clear Timeline:** Specify a clear timeline from the moment of deprecation to the removal, giving users ample time
   to prepare.
5. **Deprecated Feature Registry:** Maintain a list or registry of all deprecated features and configurations, along
   with their slated removal dates, for users to reference.

#### Summary

Introducing a structured deprecation and migration policy not only establishes trust with users but also ensures the
software evolves efficiently, balancing innovation with stability.

### Proposal 7 - Exhaustive Error Messages

#### Objective

To enhance user troubleshooting by offering comprehensive and descriptive error messages during the configuration stage,
specifically for two main types of configuration errors.

#### Rationale

Diagnosing and resolving configuration errors can be challenging without detailed insights. A more descriptive error
messaging system offers the following benefits:

1. **Parsing with Precision:**
   - **Highlight Invalid Data's Location:** Pinpoint the exact location of errors, making it easier for users to
     address.
   - **Identify Missing Fields:** Rapidly flag any necessary but absent data, speeding up the configuration process.
   - **Spot Unknown Fields:** Alert users to any unrecognized fields, helping them to quickly catch and fix
     typographical mistakes or redundant information.
2. **Clear Directions for Domain-Specific Errors:**
   - **Highlighting Semantic Issues:** Inform users about errors or warnings tied to the higher-level domain details
     specific to Iroha. This ensures that users are aware of any nuances or complexities related to Iroha itself,
     allowing for a more refined and accurate configuration.
3. **Eliminate Guesswork:**
   - **Full Transparency in Reporting:** Ensure users are immediately and fully informed of any issues, preventing
     oversights and minimizing confusion.
4. **Consistency in Communication:**
   - **Uniform, Informative Messaging:** Maintain a consistent style across error messages, bolstering clarity and user
     understanding.
5. **Streamline Multi-Error Solutions:**
   - **Bulk Reporting of Missing Fields:** If several fields are missing, display them all simultaneously for more
     efficient troubleshooting.

#### Potential Solutions & Samples

To illustrate the potential of this system, a Proof-of-Concept (PoC) repository should be created, showcasing the
clarity and utility of these enhanced error messages in Rust.

#### Summary

Implementing exhaustive error messages will drastically enhance user troubleshooting, ensuring a smoother and more
efficient configuration process for Iroha users.

#### See Also

- [[suggestion] Enhance configuration parsing with precise field locations · Issue #3470 · hyperledger/iroha](https://github.com/hyperledger/iroha/issues/3470)

### Proposal 8 - Trace Configuration Resolution

#### Objective

Implement a mechanism that displays the sequence in which configuration is determined. This allows users to grasp the
sources from which configuration values are derived, whether they're being overridden, and which resort to default
values.

#### Rationale

Transparency in configuration resolution is vital for troubleshooting and ensuring the correct setup. By providing clear
feedback on how parameters are set, users can identify potential misconfigurations or conflicts quickly.

#### Proposed Tracing Output

```
iroha-config: Loaded `~/config.toml`
  Configured:
    `iroha.public_key`
    `iroha.private_key`
    `sumeragi.block_time_ms`
    ...
iroha-config: Sourced ENV variables
  Configured:
    `torii.addr_p2p` from `API_ENDPOINT_URL` ENV var
    ...
  Overriddes:
    `sumeragi.block_time_ms` from `BLOCK_TIME_MS` ENV var
    ...
iroha-config: Final user configuration
  ... (print full configuration, including applied default values)
```

#### Enabling Configuration Tracing

To activate configuration tracing, users can use the `--trace-config` command-line argument or set the
`IROHA_TRACE_CONFIG=1` environment variable. Given the sequential nature of the configuration resolution process, the
tracing mechanism is intentionally separated from Iroha's main logger. This ensures that tracing can be performed even
before the configuration (which defines parameters like `logger.log_level`) is fully resolved. Keeping configuration
tracing as a straightforward on-off switch, without the complexity of log levels, further simplifies the user's
interaction and guarantees that all relevant trace data is captured when enabled.

#### Sensitive Information

While the trace provides detailed information, caution should be taken to avoid exposing sensitive data, such as private
keys, ensuring security isn't compromised.

#### Further Integration

Incorporating these traces into configuration-related error messages can further streamline troubleshooting, enhancing
user experience. (Refer to: [Proposal 7 - Exhaustive Error Messages](#proposal-7---exhaustive-error-messages))

#### Summary

This proposal aims to allow users to see how the config gets pieced together, making it easier to set up and fix any
hiccups along the way.

#### See Also

- [[suggestion] Trace configuration parameters resolution · Issue #3502 · hyperledger/iroha](https://github.com/hyperledger/iroha/issues/3502)

### Proposal 9 - Remove Configuration Endpoints

#### Objective

To simplify and enhance the robustness of the Iroha system by eliminating the current configuration-related endpoints.

#### Rationale

1. **Superfluous Documentation Retrieval**: Fetching configuration field documentation through the API is unnecessary
   and redundant. A well-maintained [configuration reference](#proposal-2---reference-before-implementation) should be
   the single authoritative source for all such documentation.
2. **Runtime Configuration Updates**: Allowing for run-time changes to the configuration can introduce complexity and
   potential issues that undermine system robustness. A philosophy akin to systems like Elixir/Erlang emphasizes the
   reliability of system restarts over mutable running states. In most scenarios, it's straightforward and preferable to
   restart Iroha with an updated configuration than attempt on-the-fly changes.
3. **Ambiguities in Configuration Retrieval**: The utility of fetching the entire configuration as a JSON response is
   unclear. Such an endpoint raises multiple questions:
   - Should sensitive data, such as the `private_key`, be sanitized before being returned?
   - Is there a valid use case for displaying file paths or other meta-configuration data?
   - The initial motivation for retrieving the configuration was to inspect its state after making run-time changes via
     the configuration update endpoint. However, this becomes redundant if such modifications are discouraged or
     removed.

#### Recommendation

For the reasons stated above, it's recommended to remove the configuration-related endpoints from Iroha. This step will
reduce potential points of failure, ambiguities, and complexities. Administrators should be encouraged to use the
comprehensive configuration reference for any documentation needs and to restart Iroha with an updated configuration
when changes are required.

#### Summary

This proposal advocates for the removal of configuration-related endpoints in Iroha, emphasising system simplicity,
clarity, and robustness over mutable configurations during runtime. The configuration reference will serve as the sole
authoritative documentation source, and Iroha restarts are recommended for configuration updates.

### Proposal 10 - Standardise Relative Paths Resolution

#### Objective

To define a consistent and intuitive method for resolving relative file system paths in the Iroha configuration.

#### Rationale

Currently, relative paths in the configuration file are resolved based on the current working directory (CWD), which can
lead to unpredictable results, specifically if Iroha is executed from a directory different from where the config file
is located. To address this:

1. **Config File Relative Paths**: Relative paths specified within the config file should be resolved relative to the
   location of the config file itself, not the CWD.
2. **Environment Variable Paths**: Paths set through environment variables corresponding to configuration parameters
   should be resolved relative to the CWD.
3. **Documentation**: This behaviour will be explicitly documented in the
   [configuration reference](#proposal-2---reference-before-implementation) to ensure users are aware of how relative
   paths are handled.

Standardising how relative paths are resolved can help prevent potential issues and confusion for users.

#### Summary

This proposal seeks to bring consistency and clarity to how relative file paths are resolved in Iroha's configuration.
Paths in the config file will be relative to its location, while paths from environment variables will be relative to
the CWD.

### Proposal 11 - Adopt Same Proposals for Iroha Client CLI

#### Objective

To apply the configuration improvement proposals devised for Iroha 2 to the `iroha_client_cli`.

#### Rationale

1. **Shared Foundation**: The iroha_client_cli and Iroha 2 are built on the same foundation, and the client CLI's
   configuration structure is similar to Iroha's.
2. **Consistent Experience**: Providing a consistent configuration experience between Iroha and its official client
   ensures that users and administrators familiar with one will find the other intuitive.
3. **Efficiency**: By applying the same changes and improvements to both, we can leverage the same tools, documentation
   updates, and testing procedures, leading to efficient development and maintenance.
4. **Prevent Duplication**: Ensuring both tools share the same configuration enhancements reduces the chance of
   duplicating efforts or inconsistencies in future updates.

#### Proposal Details

All proposals and changes specified for Iroha 2's configuration will be mirrored in `iroha_client_cli`. This includes
(but is not limited to):

- Adopting the same naming conventions.
- Applying the same handling for relative paths.
- Implementing the same structure and organisation for configuration fields.
- Utilising the same documentation standards and resources.

#### Summary

To ensure a consistent user experience and efficient development, the configuration improvements proposed for Iroha 2
will also be implemented in the `iroha_client_cli`. This alignment will guarantee that users and developers encounter a
uniform configuration environment across both tools.

### Proposal 12 - Handling Configuration-ISI Discrepancies

#### Objective

Ensure that the new configuration system gracefully and reliably interacts with the existing on-chain configuration set
by Iroha Special Instructions (ISI), even as the nature and implementation of these instructions are still evolving.

#### Rationale

The existing ISI mechanism allows certain configuration parameters to be set on-chain. These on-chain configurations, if
not handled correctly, could lead to inconsistencies or unexpected behaviours in nodes that might have differing local
configurations.

1. **Clarify Hierarchies**: Establish a clear priority hierarchy between local configurations and on-chain
   configurations. On-chain configurations, being backed by consensus, should take precedence over local settings.
2. **Logging & Notification**: Implement robust logging. If an on-chain configuration overrides a local setting,
   operators must be informed about it, ensuring transparency and aiding troubleshooting.
3. **Documentation**:
   - Cover the interplay between on-chain and local configurations in the new configuration documentation. Provide
     guidelines and expected behaviours for operators to refer to, ensuring they have a clear understanding of how their
     node might respond in the face of differing configurations.
   - Include a disclaimer in the documentation that the nature and behaviour of configuration ISIs are subject to change
     and may not be stable across Iroha versions. Users should utilise chain-wide configuration with caution.

#### Scope Limitation

While this proposal guides how the new configuration system should interface with the existing ISI mechanism as reliably
as possible, redesigning or altering the underlying ISI mechanism itself is beyond the scope of this proposal.

#### Summary

This proposal aims to establish clear guidelines and mechanisms for handling potential discrepancies between on-chain
and local configurations. It ensures consistent and predictable node behaviour while highlighting the evolving nature of
the ISI mechanism and advising caution in its use.

## Implementation Plan

### Step 1 - Configuration Design & Reference Creation

**Description**: This phase focuses on designing the configuration, considering user needs, best practices, and ensuring
alignment with project requirements.

**Resources & Prerequisites**:

- Comprehensive understanding of the current Iroha 2 configuration.
- Knowledge of best practices in configuration design.
- Collaboration with technical writers who will serve as the primary authors of the reference.
- Collaboration with developers and potential users to gather requirements and feedback.

**Specific Objectives**:

- Re-design naming of configuration parameters, introducing aliases for the ENV.
- Employ TOML as a domain language to describe the configuration.
- Establish a deprecation and migration policy as part of the reference.

**Deliverables**:

- A detailed configuration reference document outlining the structure, default values, data types, and descriptions for
  each configuration parameter.

**Potential Risks**:

- Misalignment with user needs if not adequately consulted.
- Overcomplication or oversimplification of configuration design.

**Feedback Loop**:

- Regularly consult with both developers and potential users to ensure design alignment with their expectations and use
  cases.

### Step 2 - Development of a Generic Configuration Library

**Description**: Develop a robust configuration library to act as the foundational piece for Iroha 2 and potentially
other projects due to its open-source nature.

**Resources & Prerequisites**:

- Might be done concurrently with [Step 1](#step-1---configuration-design--reference-creation)
- Proficient Rust developers with experience in procmacros.
- Understanding of requirements from the Configuration Design phase.

**Specific Criteria**:

- Support for exhaustive error messages, including batch-error collection and dependable deserialization failure
  messages.
- Capabilities to compose configurations from incomplete parts for merging.
- Customizable merge strategies on a field-level.
- Support for multiple configuration source formats, notably TOML.
- Consider best and worst practices used in existing configuration-related libraries (namely:
  [`config`](https://docs.rs/config/latest/config/), [`figment`](https://docs.rs/figment/latest/figment/),
  [`schematic`](https://docs.rs/schematic/latest/schematic/)).

**Deliverables**:

- A well-documented, open-source configuration library.
- Unit and integration tests to ensure functionality.

**Potential Risks**:

- The library might not cater to all specific needs of Iroha 2.
- Potential technical debt if rushed or not adequately tested.

**Feedback Loop**:

- Continuous integration and testing setup.
- Regular checks with the Configuration Design team.

### Step 3 - Iterative Implementation, Testing, and Migration Tool Development

**Description**: Implement the designed configuration reference using the developed library. Concurrently, develop a
migration tool to help users seamlessly transition from the old to the new configuration system. Ensure the
functionality of both through iterative testing.

**Resources & Prerequisites**:

- The configuration reference from [Step 1](#step-1---configuration-design--reference-creation).
- The configuration library from [Step 2](#step-2---development-of-a-generic-configuration-library).
- Testing environment and tools.

**Deliverables**:

- Fully implemented configuration for Iroha 2 according to the reference.
- A functional migration tool capable of converting older configurations to the new format.
- Comprehensive tests showcasing the configuration's functionality and adherence to the reference.

**Potential Risks**:

- Implementation might diverge from the design if not regularly checked.
- Unforeseen technical challenges or limitations.

**Feedback Loop**:

- Regular testing and validation against the configuration reference.
- Potential beta testing or feedback sessions with a subset of users to understand usability and issues with the new
  configuration and migration tool.

### Step 4 - Integration, Field Testing & Feedback Collection

**Description**: Integrate the newly developed configuration with Iroha 2. Conduct field tests to assess its real-world
performance and gather feedback.

**Resources & Prerequisites**:

- Completed and tested configuration from
  [Step 3](#step-3---iterative-implementation-testing-and-migration-tool-development).
- Access to Iroha 2 codebase and integration tools.
- A pool of testers or users for field-testing.

**Deliverables**:

- Configuration successfully integrated into Iroha 2.
- Feedback report from field tests detailing user experience, potential issues, and improvements.

**Potential Risks**:

- Integration challenges due to unforeseen incompatibilities.
- Negative user feedback or usability challenges.

**Feedback Loop**:

- Regular integration testing and validation.
- Direct communication channels with field testers to gather timely feedback.

### Step 5 - Iterative Refinement & Optimization

**Description**: After integrating the new configuration into Iroha 2, there might be some elements that need refining
based on user feedback and field-testing.

**Resources & Prerequisites**:

- Feedback from the [integration phase](#step-4---integration-field-testing--feedback-collection).
- Development and testing environments.

**Deliverables**:

- A refined configuration addressing user feedback.
- Documentation updates reflecting changes made.

**Potential Risks**:

- Over-optimization leading to reduced clarity.
- Potential rework if feedback indicates major design flaws.

**Feedback Loop**:

- Continued consultation with users.
- Performance isn't a primary concern, but the configuration should be robust and user-friendly.

## Impact Assessment

The proposed overhaul of Iroha 2's configuration system stands to significantly impact various facets of the project,
both technical and user-facing. These implications touch on the project's future scalability, user experience, and
maintainability.

### Technical Impact

- **Code Complexity**: The pursuit of an enhanced user experience may lead to a more intricate configuration codebase.
  This increased complexity might make the system harder to maintain and debug.
- **Performance Impact**: While the configuration happens at startup and isn't a recurring process, there might be a
  slight performance dip due to the enhancements. This is considered acceptable given the trade-off for improved user
  experience.

### Operational Impact

- **User Transition**: Given the main goal of this RFC — simplifying the configuration for end users — we anticipate a
  minimal migration pain for current users. The proposed changes, informed by earlier sections of this RFC, are intended
  to streamline and simplify the process for them.
- **Deployment & Migration**: The revamped configuration could bring new deployment methods or procedures. Clear
  documentation will be essential to guide system administrators and users through these changes.

### User Impact

- **Enhanced User Experience**: The main drive behind these changes is to offer users a more intuitive configuration
  mechanism, reducing barriers to adoption and use.
- **Communication & Feedback**: Regular interaction with the community, especially the early adopters, will be crucial.
  This helps to ensure that user feedback is accounted for, leading to a more refined final product.

### Documentation & Training

- **Updated Documentation**: Every change made to the configuration system should be meticulously documented. Updated
  release notes and configuration guides will serve as primary resources for early adopters.
- **Training Sessions**: While intensive tutorials might not be needed, organizing small workshops or webinars for
  significant changes can be beneficial, especially to ease any potential transition concerns.

### Financial & Time Impact

- **Resource Allocation**: The development of a new configuration library, alongside other proposed changes, will
  necessitate both time and expertise. Efficiently managing these resources is vital to maintaining the momentum of the
  project.
- **Project Timeline**: While we haven't set a specific timeline or deadline for the release, it's essential that the
  configuration overhaul doesn't become a hindrance to the broader project's progress.

### Risk Assessment

- **Anticipating Challenges**: As the project progresses, unforeseen challenges related to the proposed changes may
  arise. Having strategies to address these in advance ensures minimal disruption.
- **Backup Strategies**: Given the scope and depth of the changes, it's prudent to have contingency plans. These plans
  ensure the project continues to progress smoothly, even when faced with unexpected hurdles.

In essence, this RFC's primary objective is to drastically simplify Iroha 2's configuration for its users. Careful
planning, consistent communication, and regular feedback loops will be pivotal in achieving this objective successfully.

## Conclusion

Throughout this RFC, we've discussed fundamental enhancements for the Iroha 2 configuration system. From transitioning
to TOML and refining naming conventions, to creating an early configuration reference and clearer error messages, each
proposed change aims to elevate the user experience. Adopting these suggestions can transform Iroha 2's configuration
from a potential stumbling block to a robust foundation. We eagerly anticipate community feedback to refine these ideas
further.

[^1]:
    Please note that the configuration of keys (`public_key` and `private_key`) is a subject with its own set of
    challenges and ongoing discussions. We are evaluating options to simplify and improve this area of configuration.
    For further context, please refer to the following issues:

    - [Representation of key pair in configuration files · Issue #2135 · hyperledger/iroha](https://github.com/hyperledger/iroha/issues/2135)
    - [Move keys out of configuration · Issue #2824 · hyperledger/iroha](https://github.com/hyperledger/iroha/issues/2824)
    - [Guard against secrets leakage · Issue #3240 · hyperledger/iroha](https://github.com/hyperledger/iroha/issues/3240)

    Input on this topic is valuable.

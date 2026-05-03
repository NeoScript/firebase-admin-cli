# AGENTS.md — Notes for AI coding agents

## Project overview

**fbadmin** is a Rust CLI for Firebase Authentication administration. It wraps the `rs-firebase-admin-sdk` crate and provides subcommands for user management, custom claims, auth action links, and emulator utilities.

## Repository layout

```
src/
  main.rs           # CLI struct (clap derive), arg parsing, command dispatch
  config.rs         # Profile/config management (confy, TOML, local overrides)
  firebase.rs       # AuthBackend enum, init_firebase() — SDK initialization
  output.rs         # Render helpers: table, JSON, CSV, colored messages
  prompt.rs         # Interactive prompts (dialoguer) for missing args
  errors.rs         # IntoAnyhow trait for error-stack → anyhow conversion
  commands/
    mod.rs          # Re-exports
    users.rs        # users get/create/disable/enable/remove/list/list-inactive/count
    claims.rs       # claims get/merge/remove/clear/find
    links.rs        # links password-reset/email-verify/sign-in
    emulator.rs     # emulator clear-users/config
    config_cmd.rs   # config init/add/remove/default/list/show/which/path
    info.rs         # info — resolved connection + connectivity check
build.rs            # vergen-gitcl: embeds git SHA + build date into --version
dist-workspace.toml # cargo-dist release configuration
Cargo.toml          # Dependencies, package metadata, profiles
```

## Key patterns

- **CLI structure**: All args are defined in `main.rs` via clap derive. Each command module has a `pub async fn run(cli: &Cli, command: &XxxCommand)` entry point.
- **Error handling**: SDK calls return `Result<T, error_stack::Report<ApiClientError>>`. Use the `IntoAnyhow` trait (`.into_anyhow()`) to convert, then chain `.context("human-readable message")` for user-facing errors.
- **Interactive prompts**: When a required arg is `None`, command modules call `prompt::resolve_email()` or similar. These use `dialoguer` and are TTY-aware.
- **Output**: Always go through `output.rs` helpers (`render_single_record`, `render_table`, `render_success`, etc.) — they handle `--format` switching (table/json/csv) and colored output.
- **Config resolution**: Profile is resolved from CLI flag → `FBADMIN_PROFILE` env → `default_profile` in config. Connection merges profile settings with CLI overrides. See `config::resolve_connection()`.
- **Firebase init**: `firebase::init_firebase()` takes an `AuthBackend` enum (Emulator or Live with optional credentials/project). The `build_auth` helper in command modules wires config → AuthBackend → FirebaseAuth.
- **Logging**: `tracing` with `tracing-subscriber`. Controlled by `-v` flag count. Logs go to stderr. Use `tracing::debug!` for internal details, `tracing::info!` for notable operations.

## Build and run

```bash
cargo build                  # Debug build
cargo run -- users list      # Run with args
cargo clippy                 # Lint
cargo fmt                    # Format
```

The project requires Rust 1.94+ (edition 2024).

## Releasing

See [docs/RELEASING.md](docs/RELEASING.md) for the full release process. Short version:

1. Bump `version` in `Cargo.toml`
2. Update `RELEASES.md`
3. `git tag vX.Y.Z && git push origin main --tags`

## Dependencies of note

| Crate | Purpose |
|-------|---------|
| `rs-firebase-admin-sdk` | Firebase Auth SDK (community) |
| `google-cloud-auth` | ADC / service account auth |
| `clap` (derive) | CLI argument parsing |
| `confy` | OS-appropriate config file management |
| `dialoguer` | Interactive terminal prompts |
| `comfy-table` | Table rendering |
| `console` | TTY detection, colored/styled output |
| `anyhow` + `error-stack` | Error handling layers |
| `tracing` | Structured logging |
| `vergen-gitcl` | Build-time git metadata |

## Things to watch out for

- **`rs-firebase-admin-sdk` is community-maintained** — check for API changes when updating. There is no official Rust Firebase SDK.
- **`IntoAnyhow` trait** in `errors.rs` is the bridge between `error-stack` (used by the SDK) and `anyhow` (used everywhere else). Always use it for SDK calls.
- **`build.rs`** uses `vergen-gitcl` which shells out to `git`. CI builds need git available.
- **The `.docs/` directory** contains internal design documents and is gitignored. It's not shipped or published.
- **Config file paths** are OS-dependent (managed by `confy`). Don't hardcode paths.
- **Destructive commands** (`remove`, `clear`, `disable`) should always respect `--dry-run` and `--yes` flags.

## License

AGPL-3.0-only

# ketoy CLI

The official command-line interface for [Ketoy Cloud](https://ketoy.dev) — push SDUI screens, manage apps, and control your deployment pipeline from your terminal.

## Installation

### Install via npm (recommended)

```bash
npm install -g ketoy-dev
ketoy version
```

Installs the native `ketoy` binary for your platform (macOS / Linux / Windows, x64 & arm64). Requires **Node.js ≥ 18**. The binary is downloaded from the matching GitHub release on install (and lazily on first run if you used `--ignore-scripts`). The npm package is a thin launcher — all commands, auth, and config are handled by the native binary.

### Download binary

Download the latest release from the [Releases](https://github.com/KetoyDev/ketoy-cli-npm/releases) page for your platform:

| Platform | File |
|---|---|
| macOS (Intel) | `ketoy_<version>_darwin_amd64.tar.gz` |
| macOS (Apple Silicon) | `ketoy_<version>_darwin_arm64.tar.gz` |
| Linux (amd64) | `ketoy_<version>_linux_amd64.tar.gz` |
| Linux (arm64) | `ketoy_<version>_linux_arm64.tar.gz` |
| Windows (amd64) | `ketoy_<version>_windows_amd64.zip` |

```bash
# macOS / Linux
tar -xzf ketoy_<version>_darwin_amd64.tar.gz
sudo mv ketoy /usr/local/bin/
ketoy version
```

### Build from source

The CLI source lives in the `ketoy-cli/` directory of the KetoyVM repository:

```bash
git clone https://github.com/KetoyDev/KetoyVM
cd KetoyVM/ketoy-cli
go build -o ketoy .
ketoy version
```

Requires Go 1.21+.

## Set up a project — `ketoy init`

Add Ketoy to an existing Android project with surgical, additive, **idempotent** edits
(non-Hilt by default). Run it from your project root:

```bash
ketoy init
# or: ketoy init /path/to/AndroidProject
```

It runs step by step — re-run any time; completed steps are skipped:

1. Adds the `dev.ketoy.compiler` Gradle plugin.
2. Adds the SDK dependencies (`ketoy-bom` + runtime + annotations + capabilities + adapters).
3. Appends the `ketoy { }` build config (in-tree bundle → `assets/ketoy/main.ktx`).
4. Creates `app/ketoy-capabilities.json`.
5. Ignores private signing keys in `.gitignore`.
6. **Generates an Ed25519 signing keypair and signs the bundle by default** — the private key lands in `app/keys/release-private.key` (gitignored), the public key ships in `assets/ketoy/keys/`, and runtime signature verification is enabled in `MyApplication`. Opt out with `--no-sign`.
7. Creates `MyApplication.kt` (non-Hilt bootstrap) and links it in the manifest.
8. Wires `MainActivity` with `LocalKetoyRuntime` / `LocalKetoyBundleLoader`.
9. Adds a sample `HomeScreen.kt` (`@KetoyEntryPoint`) so `:app:ketoyBundle` produces a bundle.

**Signing is compulsory by default** — every bundle is Ed25519-signed and verified on device.
Keep `app/keys/release-private.key` out of version control (init gitignores it) and in a CI secret.
Use `--no-sign` for a quick unsigned setup (verification off).

The SDK/plugin version is fetched from `https://ketoy.dev/v1/version` (override with `--version`).

**Hilt projects:** the DI wiring differs — `init` applies only the Gradle setup and points you to
[ketoy.dev/docs](https://ketoy.dev/docs). Use `--hilt` to force this path.

| Flag | Description |
|---|---|
| `--version <v>` | Pin the Ketoy SDK/plugin version (default: latest from the version API) |
| `--hilt` | Hilt project: Gradle setup only + docs pointer |
| `--no-sign` | Skip key generation — ship an unsigned bundle (runtime verification off) |

After init: `./gradlew :app:ketoyBundle` builds the signed `.ktx`; then push it with `ketoy push ktx` (below).

## Quick Start — deploy to Ketoy Cloud

```bash
# Create an account
ketoy auth register

# Log in
ketoy auth login

# Create your developer profile
ketoy profile create

# Create an app
ketoy app create

# Push a KTX app binary
ketoy push ktx <appId> myapp.ktx --version 1

# Check your setup
ketoy doctor
```

## Commands

| Command | Description |
|---|---|
| `ketoy init [path]` | Set up Ketoy in an existing Android project (local, non-Hilt) |
| `ketoy auth register` | Create a Ketoy account |
| `ketoy auth login` | Log in and save credentials |
| `ketoy auth logout` | Remove saved credentials |
| `ketoy auth status` | Show current login state |
| `ketoy profile create` | Set up your developer profile |
| `ketoy profile show [username]` | Show your or another developer's profile |
| `ketoy profile update` | Update your profile |
| `ketoy key list` | List your API keys |
| `ketoy key revoke <keyId>` | Revoke an API key |
| `ketoy app create` | Create a new app |
| `ketoy app list` | List your apps |
| `ketoy app show <appId>` | Show app details |
| `ketoy app update <appId>` | Update app name or contacts |
| `ketoy app delete <appId>` | Delete an app |
| `ketoy namespace status <appId>` | Check namespace verification |
| `ketoy namespace verify <appId>` | Verify namespace ownership (wizard) |
| `ketoy namespace request-token <appId>` | Request DNS verification token |
| `ketoy namespace check <appId>` | Check DNS propagation |
| `ketoy push ktx <appId> <file.ktx>` | Push a KTX app binary |
| `ketoy ktx versions <appId>` | List KTX versions |
| `ketoy ktx live <appId>` | Show the currently live version |
| `ketoy ktx download <appId>` | Download a KTX version to disk |
| `ketoy ktx rollback <appId>` | Roll back KTX to an earlier version |
| `ketoy doctor` | Diagnose common issues |
| `ketoy version` | Print CLI version |

Full documentation: [REFERENCE.md](REFERENCE.md)

## Global Flags

| Flag | Description |
|---|---|
| `--json` | Output raw JSON (for CI/CD scripting) |
| `--dev` | Use development API |

## CI/CD

```bash
# In your pipeline:
APP_ID="your-app-id"
./ketoy push ktx "$APP_ID" ./build/app.ktx --version "$BUILD_NUMBER" --json
```

See [REFERENCE.md § CI/CD Integration](REFERENCE.md#15-cicd-integration) for full setup instructions.

## Shell Autocomplete

```bash
# Bash
ketoy completion bash > /etc/bash_completion.d/ketoy

# Zsh
ketoy completion zsh > "${fpath[1]}/_ketoy"

# Fish
ketoy completion fish > ~/.config/fish/completions/ketoy.fish
```

## Releasing (maintainers)

The npm package (`ketoy-dev`) is a thin launcher; the actual `ketoy` binaries are
published as GitHub **release assets** on the public
[`KetoyDev/ketoy-cli-npm`](https://github.com/KetoyDev/ketoy-cli-npm) repo. The Go
source stays in this private monorepo — only compiled binaries go public. No CI,
no goreleaser: builds are produced and pushed locally.

**One-time setup:** create the public repo `KetoyDev/ketoy-cli-npm` with at least
one commit (a README so it has a default branch), and authenticate the GitHub CLI
(`gh auth login`) as an account with write access to it.

**Each release:**

```bash
# 1. Bump the version in ketoy-cli/package.json ("version": "X.Y.Z")

# 2. Cross-build every platform and push the archives as release assets to
#    KetoyDev/ketoy-cli-npm (tag vX.Y.Z). Requires Go, tar, zip, and gh.
./ketoy-cli/scripts/release.sh

# 3. Publish the npm launcher (must be run AFTER step 2 so the binaries exist)
cd ketoy-cli && npm publish
```

The archive names produced by `release.sh` match exactly what
[`scripts/install-binary.js`](scripts/install-binary.js) requests, so
`npm install -g ketoy-dev` downloads them with no GitHub auth. To point the
downloader at a different releases repo, set `KETOY_CLI_REPO=owner/repo`.

## License

MIT

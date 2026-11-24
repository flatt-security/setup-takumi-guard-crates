<p align="center">
  <img src="branding.png" alt="Takumi Guard — a panda security guard scanning Rust crates" width="300" />
</p>

<h1 align="center">Takumi Guard for crates.io</h1>

<p align="center">
  <strong>Stop malicious Rust crates before they reach your CI.</strong><br />
  A GitHub Action that routes <code>cargo build</code> through a security proxy — no secrets, no config files, two lines of YAML.
</p>

<p align="center">
  <a href="https://github.com/flatt-security/setup-takumi-guard-crates/actions/workflows/test.yml"><img src="https://github.com/flatt-security/setup-takumi-guard-crates/actions/workflows/test.yml/badge.svg" alt="CI" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/github/license/flatt-security/setup-takumi-guard-crates" alt="License" /></a>
</p>

---

> **Not using CI?** For local setup on your laptop, see the [Takumi Guard crates.io quickstart](https://shisho.dev/docs/t/guard/quickstart/rust/).

## Contents

- [What is Takumi Guard?](#what-is-takumi-guard)
- [Quickstart (3 steps)](#quickstart)
- [Setup modes](#setup-modes)
- [Adopting Takumi Guard](#adopting-takumi-guard)
- [Inputs](#inputs)
- [Outputs](#outputs)
- [Cargo commands that talk to crates.io directly](#cargo-commands-that-talk-to-cratesio-directly)
- [Troubleshooting](#troubleshooting)
- [Security](#security)
- [Appendix: Email registration & token management](#appendix-email-registration--token-management)

---

## What is Takumi Guard?

Every `cargo build` in your CI is a trust decision. Takumi Guard sits between your workflow and crates.io, **blocking known-malicious crates before they are downloaded**.

- **How it works** -- Replaces crates.io with a sparse registry (`crates.flatt.tech`) that checks crates against a threat database in real time. Blocked crates disappear from the index, and their downloads are refused.
- **What you change** -- One step in your workflow YAML. No `Cargo.toml` edits, no secrets to manage. `Cargo.lock` keeps crates.io as the source, so the same lock file works with and without Takumi Guard.
- **What it supports** -- **Cargo 1.74 or later** (registry tokens through credential providers).

---

## Quickstart

**Goal:** Add Takumi Guard to any GitHub Actions workflow. No account required.

**Step 1.** Add the action to your workflow file (e.g. `.github/workflows/ci.yml`):

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: dtolnay/rust-toolchain@stable

  - uses: flatt-security/setup-takumi-guard-crates@v1   # <-- add this line

  - run: cargo build --locked
  - run: cargo test --locked
```

> **Ordering:** Put this action *after* the toolchain setup and *before* any Cargo command that fetches crates. It writes the source replacement to the Cargo config of the job and exports the registry token.

**Step 2.** Push the change. Every crate fetch in this job now runs through the Takumi Guard proxy, including builds from a committed `Cargo.lock`. Malicious crates are blocked automatically.

**Step 3.** *(Optional)* **Want audit logging and breach notifications?** Add a Bot ID:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # Required for authentication
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable

      - uses: flatt-security/setup-takumi-guard-crates@v1
        with:
          bot-id: "YOUR_BOT_ID"

      - run: cargo build --locked
```

> **Where do I get a Bot ID?** Create one at [Shisho Cloud byGMO](https://cloud.shisho.dev) -- or skip this entirely. Blocking works without it. The Bot ID is a public reference key, not a secret.

---

## Setup modes

| Mode | Blocks malware | Audit logging | Account needed | Best for |
|---|:---:|:---:|:---:|---|
| **[Blocking only](#blocking-only)** | Yes | No | No | OSS projects, quick evaluation |
| **[Full protection](#full-protection)** | Yes | Yes | Yes | Production workloads |
| **[Auth-only](#auth-only-advanced)** | You manage | Yes | Yes | Custom Cargo configs |

---

### Blocking only

> **No account needed.** Add one line and you are protected.

Blocks known-malicious crates. No signup, no authentication.

```yaml
- uses: flatt-security/setup-takumi-guard-crates@v1
```

The action configures the source replacement and exports the placeholder token `anonymous`, which Takumi Guard serves anonymously: known-malicious crates are blocked, but downloads are not tracked and no breach notifications are sent. Good for open-source projects or quick evaluation.

---

### Full protection

> **Recommended for production.** Blocks threats _and_ logs all crate activity to your dashboard.

```yaml
permissions:
  id-token: write

steps:
  - uses: flatt-security/setup-takumi-guard-crates@v1
    with:
      bot-id: "YOUR_BOT_ID"
```

**Key details:**
- Auth is handled via **GitHub's built-in OIDC** -- no PATs or secrets to rotate.
- The action exchanges the job's GitHub OIDC token (audience: the STS URL, which the bot's trust condition must accept) for a short-lived Takumi Guard token at job start, and exports it as `CARGO_REGISTRIES_TAKUMI_GUARD_TOKEN=Bearer <token>`. Cargo's `cargo:token` credential provider sends it with every request to the registry.
- Downloads are recorded for your organization, and breach notifications reach its configured channels.
- If authentication fails (invalid `bot-id`, missing `id-token: write`, STS unreachable), the action exits with a clear error message and the build fails. There is no silent fallback to blocking-only mode, and the registry is not configured.
- Get a Bot ID from [Shisho Cloud byGMO](https://cloud.shisho.dev).

---

### Auth-only (advanced)

> **For custom setups.** You manage the Cargo config yourself. The action only handles authentication (exporting the token).

Use this when your repository already commits a `.cargo/config.toml`, or when your Cargo config already defines `[source.crates-io]` and the default mode stops with `… already defines [source.crates-io] …`.

**Step 1.** Add the Takumi Guard registry to the Cargo config your build reads, for example the repository's `.cargo/config.toml`:

```toml
[source.crates-io]
replace-with = "takumi-guard"

[registries.takumi-guard]
index = "sparse+https://crates.flatt.tech/"
credential-provider = "cargo:token"
```

**Step 2.** Run the action with `set-registry: false` before any Cargo command:

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: actions/checkout@v4
  - uses: dtolnay/rust-toolchain@stable

  - uses: flatt-security/setup-takumi-guard-crates@v1
    with:
      bot-id: "YOUR_BOT_ID"
      set-registry: false

  - run: cargo build --locked
```

**Key details:**
- **Without Step 1, nothing goes through Takumi Guard.** The action exports the token but does not touch the Cargo config, so Cargo keeps fetching from crates.io directly, and no crate is checked or logged. The action cannot detect this.
- A committed `.cargo/config.toml` also applies outside CI. Developers who build the repository locally need their own token in `~/.cargo/credentials.toml` (see [Configure Cargo](#configure-cargo)), or Cargo fails with `no token found for takumi-guard`.
- If authentication fails, **the action exits with an error** -- there is no fallback.

---

## Adopting Takumi Guard

Cargo does not record the replacement registry in `Cargo.toml` or `Cargo.lock`. Most projects can adopt Takumi Guard with no file changes.

**For crates.io dependencies:** The action replaces crates.io with `crates.flatt.tech` in the job's Cargo config. `Cargo.lock` keeps crates.io as the source, so the same lock file works with and without Takumi Guard, and `cargo build --locked` from an existing lock file is checked against the blocklist.

**For git, path, and other-registry dependencies:** They resolve directly from where they are declared, without going through Takumi Guard. Only crates.io dependencies are checked.

**For projects with vendored dependencies:** Nothing to do. A project whose own `.cargo/config.toml` replaces crates.io with a `cargo vendor` directory keeps building from that directory, because the project's config takes precedence over the job's, and nothing is fetched. Vendored crates are therefore not checked in that job; to check them, run `cargo vendor` with Takumi Guard configured when you update the vendor directory.

> **No crates.io fallback.** With the source replacement in place, Cargo never falls back to crates.io: a registry error fails the build instead of bypassing the blocklist.

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `bot-id` | No | -- | Bot ID from Shisho Cloud byGMO. Omit for blocking-only mode. |
| `set-registry` | No | `true` | Write the source replacement to `$CARGO_HOME/config.toml`. Set to `false` if you manage the Cargo config yourself. |
| `registry-url` | No | `https://crates.flatt.tech` | Registry endpoint (must speak the Cargo sparse registry protocol). An empty value means the default. |
| `sts-url` | No | `https://sts.cloud.shisho.dev` | STS endpoint for token exchange. |
| `expires-in` | No | `1800` | Token lifetime in seconds (max 86400). |
| `audience` | No | `https://sts.cloud.shisho.dev` (the STS URL) | Audience for the OIDC token request. Override when your Bot trust condition expects a different value. |

---

## Outputs

| Output | Description |
|---|---|
| `token` | The access token for the registry. Only set when authenticated. Masked in logs. |
| `token-expires-at` | ISO 8601 timestamp of token expiration. Only set when authenticated. |

### Where the config goes

Cargo reads its user-level config from `$CARGO_HOME/config.toml`; `~/.cargo` is only the default. The action writes there, inside a block marked `# >>> setup-takumi-guard-crates >>>` that a later run of the action replaces. If the file already defines `[source.crates-io]` or `[registries.takumi-guard]` outside that block, the step fails instead of overriding it — remove yours, or use `set-registry: false`.

---

## Cargo commands that talk to crates.io directly

Commands that only download crates (`cargo build`, `fetch`, `update`, `install`, `vendor`, …) need nothing extra. Commands that use the crates.io web API stop with `crates-io is replaced with remote registry takumi-guard` because crates.io is replaced; name crates.io explicitly for them:

```bash
cargo publish --registry crates-io
cargo search serde --registry crates-io
cargo info serde --registry crates-io
cargo owner --list serde --registry crates-io
cargo yank --version 1.0.0 my-crate --registry crates-io
```

The dependencies `cargo publish` builds for verification are still fetched through Takumi Guard.

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `got 401` … `this registry requires a token: configure a cargo credential provider for it` | Cargo older than 1.74 (it cannot send registry tokens) | Use Cargo 1.74+. Run jobs on an older toolchain (e.g. an MSRV check) without this action. The action itself fails early when it finds an older Cargo. |
| `no token found for takumi-guard` | `CARGO_REGISTRIES_TAKUMI_GUARD_TOKEN` is not set in that step | Run the step after this action, in the same job. |
| `got 403` … `Takumi Guard: <crate> is blocked as a malicious package` | The crate (or that version) is on the blocklist | Remove or replace the dependency. |
| `crates-io is replaced with remote registry takumi-guard` | A crates.io web-API command | Add `--registry crates-io` (see above). |
| The step fails with `… already defines [source.crates-io] …` | Your Cargo config already replaces crates.io | Remove it, or use `set-registry: false` and add the Takumi Guard registry yourself. |
| `OIDC not available` | The job lacks `id-token: write` | Add `permissions: { id-token: write, contents: read }`. |
| `STS rejected bot-id` | The bot's trust condition does not admit this repository / workflow, or its audience is not the STS URL | Check the bot's trust conditions in Shisho Cloud byGMO, or set `audience`. |

---

## Security

- **Short-lived tokens** -- 30 minutes by default, 24 hours max.
- **Auto-masked** -- Tokens are automatically masked in workflow logs.
- **Job-scoped token** -- The action exports the token as `CARGO_REGISTRIES_TAKUMI_GUARD_TOKEN` through `$GITHUB_ENV` on the ephemeral runner. It is not written to `config.toml`, `credentials.toml`, or any file tracked by git.
- **Host-bound credential** -- Cargo's `cargo:token` credential provider sends the token only to the `takumi-guard` registry, as `Authorization: Bearer <token>` over HTTPS. Other registries never receive it.
- **No crate bytes through us** -- After the blocklist check, `.crate` downloads are served via 302 redirect to `static.crates.io`. Crate bytes never traverse our infrastructure, and Cargo still verifies each one against the checksum in the index.
- **No direct crates.io fallback** -- With the source replacement in place, crates.io dependencies can only be fetched through the proxy.

---

## Appendix: Email registration & token management

> **Optional.** Register your email to receive breach notifications if a crate you downloaded is later flagged as malicious. This works for local development -- CI workflows should use [Full protection](#full-protection) instead.

### Register

```bash
curl -X POST https://crates.flatt.tech/api/v1/tokens \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com"}'
```

Check your inbox and click the verification link. You will receive a token like `tg_anon_xxx...`.

**Language preference:** Add `"language": "ja"` to receive emails in Japanese. Defaults to English (`"en"`) if omitted.

```bash
curl -X POST https://crates.flatt.tech/api/v1/tokens \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "language": "ja"}'
```

> **Reusing an existing token:** If you have already registered with Takumi Guard for npm, PyPI, RubyGems, Go, or Packagist, the same `tg_anon_*` token works here -- it is a universal key across all ecosystems.

### Configure Cargo

Requires Cargo 1.74 or later. Add the source replacement to `~/.cargo/config.toml` (or `$CARGO_HOME/config.toml`):

```toml
[source.crates-io]
replace-with = "takumi-guard"

[registries.takumi-guard]
index = "sparse+https://crates.flatt.tech/"
credential-provider = "cargo:token"
```

Then attach your token in `~/.cargo/credentials.toml`. The value includes the `Bearer` scheme:

```toml
[registries.takumi-guard]
token = "Bearer tg_anon_xxx..."
```

After this, every `cargo build`, `cargo fetch`, and `cargo install` routes through Takumi Guard with your identity attached. If a crate you downloaded is later found to be malicious, you will receive a breach notification email.

### Check token status

```bash
curl -H "Authorization: Bearer tg_anon_xxx..." \
  https://crates.flatt.tech/api/v1/tokens/status
```

### Rotate your key

```bash
curl -X POST -H "Authorization: Bearer tg_anon_xxx..." \
  https://crates.flatt.tech/api/v1/tokens/regenerate
```

Returns a new API key. The old one is invalidated immediately. Update your `credentials.toml` with the new key.

### Revoke a token

```bash
curl -X DELETE -H "Authorization: Bearer tg_anon_xxx..." \
  https://crates.flatt.tech/api/v1/tokens
```

---

<p align="center">
  Built by <a href="https://flatt.tech">GMO Flatt Security Inc.</a><br />
  <a href="LICENSE">MIT License</a>
</p>

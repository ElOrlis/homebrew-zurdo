# homebrew-zurdo

Homebrew tap for **Zurdo** — a CLI that drives LLM agents through a PRD's tasks
with independent verification. Zurdo is proprietary software; this tap installs
the licensed pre-built binary.

## Install

```sh
brew install ElOrlis/zurdo/zurdo
```

The formula pulls a pre-built binary from the public
[zurdo-dist](https://github.com/ElOrlis/zurdo-dist) releases — no toolchain
required. Shipped targets: macOS Apple Silicon, and Linux x86_64 / aarch64.

## Upgrading from a pre-1.0 install

Older versions were tapped with `brew tap ElOrlis/zurdo https://github.com/ElOrlis/zurdo`,
which points at the source repository. That repo is now private and no longer
carries the formula, so a one-time re-tap against this public tap is needed:

```sh
brew untap elorlis/zurdo
brew install ElOrlis/zurdo/zurdo
```

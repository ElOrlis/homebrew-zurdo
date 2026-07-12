# Homebrew tap for Zurdo

> A Rust CLI that drives LLM agents through a PRD's tasks in a loop — and **independently verifies** every acceptance criterion instead of trusting the agent's self-report.

This is the official Homebrew tap for [Zurdo](https://zurdo.numeron.ai). The formula installs a licensed pre-built binary from the public [zurdo-dist](https://github.com/ElOrlis/zurdo-dist) releases — no toolchain required. Zurdo is proprietary, closed-source software.

## Install

```sh
brew install ElOrlis/zurdo/zurdo
```

Shipped targets: macOS Apple Silicon, and Linux x86_64 / aarch64 via [Homebrew on Linux](https://docs.brew.sh/Homebrew-on-Linux). (Intel macOS is not pre-built yet.)

> **Upgrading from a pre-1.0 install?** Older versions were tapped with
> `brew tap ElOrlis/zurdo https://github.com/ElOrlis/zurdo` — pointing at the
> source repo, which is now private and no longer carries the formula. Re-tap
> once against this public tap:
>
> ```sh
> brew untap elorlis/zurdo
> brew install ElOrlis/zurdo/zurdo
> ```

Zurdo shells out to an agent CLI for the executor role — install and authenticate at least one of [`claude`](https://docs.claude.com/en/docs/claude-code/overview), [`codex`](https://github.com/openai/codex), or [`copilot`](https://github.com/github/gh-copilot). See the [installation guide](https://zurdo.numeron.ai/docs/installation) for details, including release-tarball installs with checksum verification.

## Documentation

Full documentation lives at **[zurdo.numeron.ai](https://zurdo.numeron.ai)**:

- [How it works](https://zurdo.numeron.ai/docs/how-it-works) — the verification loop, state model, evidence integrity, crash recovery
- [Usage](https://zurdo.numeron.ai/docs/usage) — everyday workflows, run output, CI integration, troubleshooting
- [Writing PRDs](https://zurdo.numeron.ai/docs/writing-prds) — the PRD grammar and the [hints reference](https://zurdo.numeron.ai/docs/hints)
- [Commands](https://zurdo.numeron.ai/docs/commands) — the complete CLI reference and exit codes
- [Configuration](https://zurdo.numeron.ai/docs/configuration) and [Providers](https://zurdo.numeron.ai/docs/providers)
- [Roadmap](https://zurdo.numeron.ai/docs/roadmap) — what's shipping next

Release notes: [zurdo-dist releases](https://github.com/ElOrlis/zurdo-dist/releases). Bug reports and feature requests: [zurdo-dist issues](https://github.com/ElOrlis/zurdo-dist/issues).

## Quick start

```sh
zurdo init                     # write .zurdo/config.toml, install bundled skills
zurdo validate prds/hello.md   # grammar check, free and instant
zurdo run prds/hello.md        # drive the loop
zurdo report prds/hello.md     # inspect what happened
```

See the [quick start](https://zurdo.numeron.ai/#quick-start) for a complete worked example.

## License

Proprietary — Copyright © 2026 Numeron Technologies Inc. Zurdo is an independent reimplementation inspired by [paullovvik/ralph-loop](https://github.com/paullovvik/ralph-loop) (MIT).

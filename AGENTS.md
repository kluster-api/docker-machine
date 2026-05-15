# AGENTS.md

This file provides guidance to coding agents (e.g. Claude Code, claude.ai/code) when working with code in this repository.

## Repository purpose

This is the AppsCode/`kompute` mirror of [Rancher Machine](https://github.com/rancher/machine) — which is itself a fork of [Docker Machine](https://github.com/docker/machine). It lets you provision Docker hosts on cloud providers (AWS EC2, Azure, GCP, DigitalOcean, etc.) and on local virtualization (HyperV, KVM, etc.). The CLI is the `rancher-machine` binary; this repo is consumed both as a binary and as a Go library by the AppsCode `docker-machine-operator`.

The Go module path is unchanged from upstream: **`github.com/rancher/machine`**. The git remote is `git@github.com:kompute/machine.git`. Locally placed under `go.klusters.dev/docker-machine`.

## Architecture (upstream Rancher Machine 2.0 layout)

- `cmd/rancher-machine/` — entry point (`machine.go`, `machine_test.go`, `setup_test.go`).
- `commands/` — CLI subcommands (`create`, `active`, `config`, `env`, …). Each subcommand has a matching `_test.go`. `commandstest/` holds shared test helpers.
- `drivers/` — one subdirectory per provider plugin: `amazonec2`, `azure`, `digitalocean`, `exoscale`, `generic`, `google`, `hyperv`, `vmwarefusion`, `virtualbox`, etc. Plus `driverutil/`, `errdriver/`, and `fakedriver/` for tests.
- `libmachine/` — the embeddable library API (`libmachine.go` is the front door). Subpackages: `auth/`, `cert/`, `check/`, `crashreport/`, `drivers/`, `engine/`, `host/`, `hosttest/`, etc.
- `version/` — version variables injected via `-ldflags`. Build scripts set `github.com/rancher/machine/version.{Version,GitCommit}`.
- `scripts/` — the actual Make targets (`build`, `ci`, `clean`, `entry`, `package`, `release`, `test`, `validate`, `version`). The Makefile only delegates to these via dapper.
- `Dockerfile.dapper` — Rancher's [dapper](https://github.com/rancher/dapper) build container (SUSE BCI Go 1.20).
- `Dockerfile` — produced binary image.
- `Makefile` + `Makefile.inc` — thin wrappers that call `./scripts/<target>` inside dapper.
- `package/` — packaging metadata for releases.
- `experimental/`, `its/`, `test/`, `contrib/` — exactly what they sound like.

## Common commands

This repo uses Rancher's **dapper** wrapper, not the AppsCode Docker harness. The Makefile auto-downloads `.dapper` on first run (except on Apple Silicon — see the caveat below).

- `make` — default target `ci`: runs `clean build test package` inside the dapper container.
- `make build` — `scripts/build`. With `CROSS=1`, builds a fat set of platform-specific binaries (`bin/rancher-machine-Linux-x86_64`, `…-Darwin-x86_64`, `…-Windows-x86_64.exe`, plus arm/arm64/armv6l/armv7l/ppc64le/s390x). Without `CROSS`, just builds `bin/rancher-machine` for the host. Either way it injects the version into `github.com/rancher/machine/version.{Version,GitCommit}` via `-ldflags`.
- `make test` — `scripts/test`.
- `make validate` — `scripts/validate` (linters).
- `make package` — `scripts/package`.
- `make release` — `scripts/release` (used by CI).
- `make version` — print the resolved version.
- `make clean` — `scripts/clean`.

> **Apple Silicon (darwin/arm64) caveat**: the Makefile refuses to download dapper. You either build dapper yourself and place it as `.dapper`, or skip dapper and run the scripts directly with a local Go toolchain:
>
> ```
> ./scripts/version  # source the version env
> go build ./cmd/rancher-machine
> ```

Run a single Go test (local toolchain):

```
go test ./drivers/amazonec2/... -run TestName -v
```

## Conventions

- Module path is `github.com/rancher/machine` (**upstream, unchanged**); do not rename. Imports must use that, not `kompute/machine` or `go.klusters.dev/docker-machine`.
- This is a **track-upstream** fork. Prefer rebasing onto upstream Rancher Machine rather than diverging; isolate any AppsCode-specific changes so they can be replayed on top of an upstream sync. Note that origin/master has been force-pushed in the past — be careful with local commits.
- License: Apache-2.0 (`LICENSE`).
- Sign off commits (`git commit -s`).
- Issues are tracked **upstream** at https://github.com/rancher/rancher/issues per the README — keep that contract intact when filing/closing issues from this fork.
- The build system is intentionally Rancher's dapper flow; don't introduce a parallel AppsCode-style Makefile. Add new targets as `scripts/<name>` files and let the Makefile's `$(TARGETS)` rule pick them up.
- New driver plugins go under `drivers/<provider>/` matching the existing layout (`Name()`, `Create()`, `Remove()`, etc., implementing the `libmachine/drivers.Driver` interface).
- Cross-compile expects `CGO_ENABLED=0`, `-tags netgo`, and a static `-extldflags` — keep new code free of cgo.

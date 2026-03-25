# Technical Decisions

## Docker action vs composite action

ShieldCI uses a Docker action instead of a composite action because:
- The Go binary needs to be compiled — composite actions only support shell scripts and other actions
- Docker gives a fully reproducible environment with pinned dependencies
- The binary can be tested locally with `act`

## `text/template` vs external templating engine

Go's standard `text/template` was chosen over alternatives (Jinja2, Handlebars, etc.) because:
- Zero external dependencies for the templating itself
- Ships with Go — no extra `go get`
- `embed.FS` + `text/template` gives a single self-contained binary with templates baked in

## `embed.FS` for templates

Templates are embedded into the binary at compile time using `//go:embed`. This means:
- The Docker image only needs the compiled binary — no need to copy template files separately
- Templates can't be accidentally missing at runtime

## GitHub API via `google/go-github`

The official Go client for the GitHub API was chosen because:
- Typed structs for all API responses — no manual JSON parsing
- Actively maintained by Google
- Full coverage of the Git Data API needed for branch/commit/PR creation

## Input mapping in `action.yml`

Docker actions do NOT automatically expose inputs as `INPUT_*` environment variables (unlike JavaScript actions). The `env:` block under `runs:` is mandatory to bridge inputs to the container.

## Output via `$GITHUB_OUTPUT`

The deprecated `::set-output::` workflow command is ignored on current runners. All outputs are written by appending `key=value` to the file at `$GITHUB_OUTPUT`.

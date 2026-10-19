# hook-exec

[![Latest Release](https://img.shields.io/github/v/release/SemRels/hook-exec?label=version&color=blue)](https://github.com/SemRels/hook-exec/releases/latest)
[![CI](https://github.com/SemRels/hook-exec/actions/workflows/ci.yml/badge.svg)](https://github.com/SemRels/hook-exec/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/SemRels/hook-exec)](LICENSE)
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/SemRels/hook-exec/badge)](https://scorecard.dev/viewer/?uri=github.com/SemRels/hook-exec)
[![REUSE status](https://api.reuse.software/badge/github.com/SemRels/hook-exec)](https://api.reuse.software/info/github.com/SemRels/hook-exec)

Run a user-controlled shell command in any external-plugin phase supported by
semrel: `condition`, `generator`, `pre-tag`, or `release`.

## Recommended configuration

Use an explicit phase and `cmd`:

```yaml
plugins:
  - uses: exec
    phase: pre-tag
    args:
      cmd: ./scripts/build.sh
      timeout: 10m
      env: |
        BUILD_MODE: release
```

Release context is already available to the child process through environment
variables, so prefer this form:

```sh
./scripts/build.sh "$SEMREL_VERSION"
```

instead of inserting values into the shell command.

## Arguments

| Argument | Environment variable | Description |
|---|---|---|
| `cmd` | `SEMREL_PLUGIN_CMD` | Command for the configured semrel phase |
| `shell` | `SEMREL_PLUGIN_SHELL` | Shell executable; defaults to `sh` or `cmd` on Windows |
| `cwd` | `SEMREL_PLUGIN_CWD` | Child working directory |
| `env` | `SEMREL_PLUGIN_ENV` | YAML or JSON string map merged into the child environment |
| `timeout` | `SEMREL_PLUGIN_TIMEOUT` | Go duration such as `30s` or `10m`; default `5m` |
| `runInDryRun` | `SEMREL_PLUGIN_RUN_IN_DRY_RUN` | Execute even when `SEMREL_DRY_RUN=true` |
| `phase` | `SEMREL_PLUGIN_PHASE` | Explicit phase for lifecycle-keyed commands |

The plugin invokes POSIX shells with `-c`, PowerShell with
`-NoProfile -NonInteractive -Command`, and `cmd.exe` with `/D /S /C`.
Non-zero exit codes and timeouts fail the plugin. On Unix, timeout cancellation
terminates the shell process group so descendants do not continue running.

`env` must be a string because semrel maps each `args` value to one environment
variable:

```yaml
args:
  env: |
    FOO: bar
    API_MODE: release
```

## Lifecycle-keyed commands

For compatibility with `@semantic-release/exec` terminology, the following
keys can be used instead of `cmd`:

| Argument | semrel phase | Behavior |
|---|---|---|
| `verifyConditionsCmd` | `condition` | Gate the release |
| `generateNotesCmd` | `generator` | Stdout becomes `SEMREL_CHANGELOG`; diagnostics belong on stderr |
| `verifyReleaseCmd` | `pre-tag` | Runs before `prepareCmd` |
| `prepareCmd` | `pre-tag` | Prepare build or generated files |
| `publishCmd` | `release` | Runs before `successCmd` |
| `successCmd` | `release` | Post-publish success action |

Set `args.phase` when the configuration contains commands for more than one
phase. If exactly one phase group is present, hook-exec infers it.

semrel currently has no external `analyzer` phase and does not invoke plugins
after a failed pipeline. Consequently, `analyzeCommitsCmd` and `failCmd` are
rejected with an explicit error rather than silently ignored.

## Output contract

- Generator commands write generated notes to stdout. Core semrel captures that
  stream and passes it to later plugins as `SEMREL_CHANGELOG`.
- All other phases stream stdout and stderr to the user.
- Plugin diagnostics and the schema handshake always use stderr.

## Release context and templates

The child inherits `SEMREL_VERSION`, `SEMREL_CURRENT_VERSION`,
`SEMREL_TAG_NAME`, `SEMREL_BRANCH`, `SEMREL_BUMP`, `SEMREL_CHANGELOG`,
`SEMREL_COMMITS`, and the other context values supplied by semrel.

Commands also support standalone `${nextRelease.version}`,
`${lastRelease.version}`, `${gitTag}`, `${tag}`, `${channel}`, `${branch}`, and
`${date}` placeholders. hook-exec shell-quotes inserted values. Do not wrap a
placeholder in additional quotes; use the corresponding `SEMREL_*` environment
variable whenever possible.

## Dry run

When `SEMREL_DRY_RUN=true`, commands are logged and skipped by default. Set
`runInDryRun: "true"` only for commands that are known to be read-only.

## Security model

hook-exec intentionally executes trusted repository configuration with the
permissions of the semrel process. Review command changes like CI workflow
changes.

- Never concatenate commit messages, changelog content, contributor data, or
  branch names into a command.
- Consume release context through quoted environment variables in a script.
- Do not place secrets directly in `.semrel.yaml`; provide them through the CI
  environment.
- Use a minimal shell and working directory, and set a finite timeout.

## Local development

```bash
go test ./...
go build ./cmd/plugin
```

## License

Apache-2.0

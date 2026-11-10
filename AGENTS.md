# AGENTS.md

Canonical home of the paho MQTT client code shared by the weewx-mqtt extensions. The extensions
vendor copies of `bin/user/mqttcommon.py`; this repository is the only place that code is edited.

## Role & boundaries (the vendoring contract)

- `bin/user/mqttcommon.py` and `bin/user/tests/unit/test_mqttcommon.py` must stay
  **byte-identical** in this repository and in publish, subscribe and replicate.
- Never edit a consumer's copy; an installed station also carries copies (e.g. `weewx-data`),
  which are replaced by installs, not synced.
- Edit here, then from each consumer root:

  ```bash
  source .devenv
  commonrepos.sh          # copy the common-files.txt entries in
  commonrepos.sh --check  # report the ones that differ
  ```

  `commonrepos.sh` lives in development-tools and resolves this repository as its sibling
  (`WEEWX_COMMON_SRC` overrides the parent directory). The consumer's `.devenv` names this
  repository in `WEEWX_COMMON_REPOS`.
- The consumers' analyze-test workflow compares their copies against `common-ref` (currently `v1`),
  so a stale copy fails the build.
- Consumers are independent installs, which is why the copies exist at all: an extension installs
  from its repository archive and writes `bin/user/mqttcommon.py`.

## Layout

- `bin/user/mqttcommon.py` — api variant selection (`select_api`, `parse_protocol`), client
  creation, connect and asynchronous connect, credentials, TLS configuration, reconnect delays,
  callback adaptation, paho log levels, the `Logger`/`LoggerThread` logging skeleton (the log
  queue and its drain loop), and the `MessageThrottle` log-throttling runtime. Imports weewx
  lazily, inside `Logger._configure` and `log_environment`.
- `bin/user/tests/unit/test_mqttcommon.py` — stubbed-paho unit tests. Also vendored, so changes
  here have to be copied out like the module.
- `common-files.txt` — the manifest of files consumers copy in, one path per line, read by
  development-tools at the ref the consumer declares. **A new shared file that is not listed here
  never reaches a consumer.**
- `README.md` — the user-facing statement of the contract. This repository is the one exception
  to the docs-first README policy of the consumers: it has no `docs/` site, so its README lives at
  the root. Keep it and this file in step.

## Environment

Run everything from the repo root:

```bash
source .devenv
export PYTHONPATH="$WEEWX_PYTHONPATH"   # == bin, or bin:../weewx/src
```

- `.devenv` defines `WEEWX_MODULE_NAME`, `WEEWX_PYENV_VERSION`, `WEEWX_TOOL_PATH`,
  `WEEWX_LINT_PATHS`, `WEEWX_TEST_SUITES`, and puts the helper scripts on `PATH`. It does not set
  `WEEWX_COMMON_REPOS` — this repository is the shared source, not a consumer.
- WeeWX comes from PyPI when it is installed, otherwise from the sibling checkout `../weewx/src`
  (`.devenv` detects which).
- The full pre-commit gate is `~/weewx_dev/weewx-extensions/development-tools/tools/verify.sh`
  (environment, lint, actionlint, shared-config and common-repo checks, all test suites).

## Commands

```bash
source .devenv
export PYTHONPATH="$WEEWX_PYTHONPATH"

# All unit tests. No broker or network: paho is stubbed.
pytest bin/user/tests/unit

# pylint needs weewx importable even though the module imports it lazily.
pylint ./bin/user/mqttcommon.py -r n
pylint ./bin/user/tests -r n --rcfile .pylintrc-tests

# flake8, max-line-length 145 (config: .flake8)
flake8 ./bin/user/mqttcommon.py ./bin/user/tests

# The full pre-commit gate: environment, lint, actionlint, shared-config and
# common-repo checks, all suites.
~/weewx_dev/weewx-extensions/development-tools/tools/verify.sh
```

## Architecture / behavior

- **paho v1 and v2 must both work.** `select_api` picks the variant from the protocol, and
  `has_v2_api()` gates the v2 paths. The v1-only CI job lints with `continue-on-error` because
  pylint reports false E1101/E1123 against 1.6.1.
- **Python 3.9 syntax** — the CI matrix starts at 3.9.13.
- Keep the module free of new weewx imports at module scope; the consumers' unit suites import it
  without a WeeWX install.

## Testing conventions and quirks

- The `Logger` message format and the entry dicts are part of the consumers' tests. Changing a
  rendered line or an entry key means the copies' tests change too — same round.
- The tests are vendored along with the module (`common-files.txt`), so a test change is a
  cross-repo change: run the suite here, then `commonrepos.sh` into each consumer.
- Tests run under pytest or directly via `unittest.main()`. Because this file is vendored, its
  runner is the one the consumers inherit; do not add a `helpers.run_tests()` dependency.

## Versioning and release (MQTTCOMMON_API)

There is no `VERSION` here; consumers bump their own, in two files each.

`MQTTCOMMON_API = (1, 1)` in `mqttcommon.py` is the station-level guard: extensions are installed
independently, so an older one can overwrite a newer module. Each consumer checks it at import
time and raises when its copy is too old:

```python
if getattr(mqttcommon, 'MQTTCOMMON_API', (0, 0)) < (1, 1):
```

Bump it only when a change requires consumers to have a newer copy (a new call or behaviour they
now use) — not for wording or internal fixes. One bump is four edits in the same round:

1. the tuple in `bin/user/mqttcommon.py`;
2. the assert in `bin/user/tests/unit/test_mqttcommon.py`;
3. the import-time check in `mqttpublish.py`, `mqttsubscribe.py`, `mqttreplicate.py`.

It was `(1, 0)` until `connect_async` was added; consumers now require `(1, 1)` for the shared
connect surface.

## CI

`analyze-test.yml` is local rather than a call to the development-tools reusable workflow because
`mqttcommon` is not an installable extension. It needs a single WeeWX install only so pylint can
resolve the lazily imported names, it runs the unit suite only (no broker, no func/integ/e2e), and
it has no `common-repos` to sync — it is the source. The consumers' wrapper is extension-shaped:
it would add broker setup, the absent suites and the WeeWX-version matrix, and lose PR coverage.

The workflow stays the locally-runnable equivalent of `verify.sh` so the two cannot drift: the
Python 3.9.13–3.14 matrix the extensions themselves support (pylint, flake8, unit tests,
coverage), plus a `paho-mqtt==1.6.1` job (report-only lint; see Architecture / behavior), because
both paho generations must keep working in the consumers. Releases are tagged by the shared
`versioning.yml@v6` the consumers use.

## Commits

Never commit, push, tag or release unless explicitly asked. A request to make a change is not
permission to commit it, and "do it" does not include committing. Propose the commit message and
wait for the go-ahead. Don't pre-stage; staged files are not a commit request (the user stages
while reviewing). Once approved, stage and commit that step in one go.

## Cross-repo workflow

Single changes stay in this repository. Cross-repo work (a shared change, rename, wording pass)
starts in the workspace root (`weewx-mqtt`): see the workspace `AGENTS.md` for the order of a
cross-repo change and `./status-all.sh` to see every repository's state at once.
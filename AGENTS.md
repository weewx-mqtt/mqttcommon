# AGENTS.md

Canonical home of the paho MQTT client code shared by the weewx-mqtt extensions. The extensions
vendor copies of `bin/user/mqttcommon.py`; this repository is the only place that code is edited.

## Layout

- `bin/user/mqttcommon.py` — api variant selection (`select_api`, `parse_protocol`), client
  creation, connect, credentials, TLS configuration, reconnect delays, callback adaptation, paho
  log levels, the `Logger`/`LoggerThread` logging skeleton (the log queue and its drain loop), and
  the `MessageThrottle` log-throttling runtime. Imports weewx lazily, inside `Logger._configure`
  and `log_environment`.
- `bin/user/tests/unit/test_mqttcommon.py` — stubbed-paho unit tests. Also vendored, so changes
  here have to be copied out like the module.
- `common-files.txt` — the manifest of files consumers copy in, one path per line, read by
  development-tools at the ref the consumer declares. **A new shared file that is not listed here
  never reaches a consumer.**
- `README.md` — the user-facing statement of the contract. Keep it and this file in step.

## The vendoring contract

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

## MQTTCOMMON_API

`MQTTCOMMON_API = (1, 0)` in `mqttcommon.py` is the station-level guard: extensions are installed
independently, so an older one can overwrite a newer module. Each consumer checks it at import
time and raises when its copy is too old:

```python
if getattr(mqttcommon, 'MQTTCOMMON_API', (0, 0)) < (1, 0):
```

Bump it only when a change requires consumers to have a newer copy (a new call or behaviour they
now use) — not for wording or internal fixes. One bump is four edits in the same round:

1. the tuple in `bin/user/mqttcommon.py`;
2. the assert in `bin/user/tests/unit/test_mqttcommon.py`;
3. the import-time check in `mqttpublish.py`, `mqttsubscribe.py`, `mqttreplicate.py`.

It starts at `(1, 0)` and has not been bumped — nothing has been released.

## Commands

```bash
source .devenv
export PYTHONPATH="$WEEWX_PYTHONPATH"   # == bin, or bin:../weewx/src

# All unit tests (95, well under a second). No broker or network: paho is stubbed.
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

## Gotchas

- **paho v1 and v2 must both work.** `select_api` picks the variant from the protocol, and
  `has_v2_api()` gates the v2 paths. The v1-only CI job lints with `continue-on-error` because
  pylint reports false E1101/E1123 against 1.6.1.
- **Python 3.9 syntax** — the CI matrix starts at 3.9.13.
- Keep the module free of new weewx imports at module scope; the consumers' unit suites import it
  without a WeeWX install.
- The `Logger` message format and the entry dicts are part of the consumers' tests. Changing a
  rendered line or an entry key means the copies' tests change too — same round.
- There is no `VERSION` here; consumers bump their own, in two files each.

## CI

`.github/workflows/analyze-test.yml` runs locally-runnable equivalents of `verify.sh`: the Python
3.9.13–3.14 matrix (pylint, flake8, unit tests, coverage) plus a `paho-mqtt==1.6.1` job.
`versioning.yml` is the reusable tagger called on release.

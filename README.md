# mqttcommon

The MQTT client code shared by the WeeWX extensions
[subscribe](https://github.com/weewx-mqtt/subscribe),
[publish](https://github.com/weewx-mqtt/publish), and
[replicate](https://github.com/weewx-mqtt/replicate).

It is not installed on its own. Each extension ships a copy of `bin/user/mqttcommon.py` (its
`install.py` lists it) because a WeeWX extension installs from the repository archive, and all three
extensions write the same `bin/user/mqttcommon.py` on a station.

## Consumers

Do not edit the copy in an extension. Change the file here and pull it in:

```bash
mqttcommon.sh           # copy bin/user/mqttcommon.py and its tests in
mqttcommon.sh --check   # report the ones that differ
```

`mqttcommon.sh` lives in
[development-tools](https://github.com/weewx-extensions/development-tools/tree/master/tools) and
resolves this repository as its sibling (`MQTTCOMMON_SRC` overrides the path). The analyze-test
workflow compares the extension's copies against `common-ref` (default `master`), so a stale copy
fails the build.

## MQTTCOMMON_API

Consumers verify `MQTTCOMMON_API` at import time and fail loudly when their copy is older than they
require. It is the station-level guard: extensions are installed independently, so an older
extension can overwrite a newer module. It has not been bumped yet — nothing has been released.

## Development

```bash
source .devenv
pytest bin/user/tests/unit
```

The module imports weewx lazily (inside `Logger._configure` and `log_environment`), so it stays
importable without a WeeWX install, but pylint needs weewx importable to resolve those imports.

# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.1]

- Add `connect_async` to the shared MQTT client code; the `MQTTCOMMON_API` guard moves from `(1, 0)` to `(1, 1)`.
- Support TLS 1.2 on builds that do not provide the `PROTOCOL_TLSv1_2` constant.

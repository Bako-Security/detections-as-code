# detections-as-code

Detection content for a Splunk + IRIS DFIR home lab, managed as code.

This repository holds **only the detections**: SPL, per-detection metadata, tests and docs.
The integration and deployment tooling lives in a separate private repository.

## Layout

| Path          | Purpose                                                        |
|---------------|----------------------------------------------------------------|
| `detections/` | One YAML file per detection (search, schedule, severity, tags) |
| `tests/`      | Validation for the detection files, run in CI                  |
| `docs/`       | Schema description, conventions, design notes                  |

## Status

Early scaffolding. The detection YAML schema is not finalised yet.

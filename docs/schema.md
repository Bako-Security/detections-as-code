# Detection YAML schema (draft, v1)

One YAML file per detection under `detections/<platform>/<id>.yml`.
The file is the source of truth. The private pipeline builds Splunk
`savedsearches.conf` and the per-detection IRIS metadata from it.

This repo is public, so a detection file holds detection *logic and content* only.
Environment specifics (IRIS customer ids, hostnames, API keys, numeric severity ids)
live in the private pipeline repo.

Start from [detection_template.yml](detection_template.yml).

## Fields

| Field | Req | Type | Notes |
|-------|-----|------|-------|
| `schema_version` | yes | int | Bump on breaking schema changes. |
| `id` | yes | string | `TXXXX` or `TXXXX.XXX`, optional `-slug` suffix. Filename must match. Becomes the Splunk saved search name. |
| `status` | yes | enum | `experimental`, `test`, `production`, `deprecated`. Only `production` deploys enabled. |
| `author` | yes | string | |
| `created` / `modified` | yes | date | ISO 8601. |
| `version` | yes | int | Increment on any logic change. |
| `environment` | yes | enum | `homelab` or `attack-range`. Mapped to an IRIS customer by the pipeline. |
| `attack.tactics` | no | list | MITRE ATT&CK tactic slugs. |
| `attack.techniques` | no | list | Technique IDs. Must include the technique in `id`. |
| `data_sources` | yes | list | Human-readable log sources the search depends on. |
| `search` | yes | string | The SPL. |
| `schedule.cron` | yes | string | Standard 5-field cron. |
| `schedule.earliest` / `latest` | yes | string | Splunk relative time modifiers. |
| `trigger` | no | object | `type`, `comparator`, `threshold`. Default: results > 0. |
| `throttle.period` / `fields` | no | string / list | Suppression. Needed when the search window overlaps the schedule. |
| `severity` | yes | enum | `informational`, `low`, `medium`, `high`, `critical`. |
| `alert.title` | yes | string | Static text shown in IRIS. Convention: ends with `(<id>)`. |
| `alert.description` | yes | string | Shown in IRIS. Written for the triaging analyst. |
| `alert.tags` | yes | list | Convention: `[<id>, <data source>, <environment>]`. |
| `triage.*`, `references` | no | | Not consumed by the pipeline yet. |

## How fields reach IRIS

The pipeline writes the IRIS metadata onto the Splunk saved search itself, as
alert action parameters. There is no sidecar metadata file: the search and its
metadata are one object, so they cannot drift apart, and deploying a detection
is a single REST call that needs no Splunk restart and no access to the Splunk
host.

| Saved search key | Source |
|------------------|--------|
| `action.send_to_iris.param.alert_title` | `alert.title` |
| `action.send_to_iris.param.alert_description` | `alert.description` |
| `action.send_to_iris.param.severity_id` | `severity`: informational=2, low=3, medium=4, high=5, critical=6 |
| `action.send_to_iris.param.customer_id` | `environment` (mapping kept in the private repo) |
| `action.send_to_iris.param.tags` | `alert.tags` joined with `,` (no spaces) |
| `action.send_to_iris.param.detection_id` / `detection_version` | `id` / `version`, as provenance |

Because a tag list is comma-joined, a tag may not itself contain a comma. The
pipeline rejects one that does.

## What the pipeline sets on the Splunk side

Generated from the detection: `search`, `cron_schedule`,
`dispatch.earliest_time` / `latest_time`, the alert trigger, suppression from
`throttle`, `alert.severity`, and `disabled` from `status` (only `production`
deploys enabled).

Set on every detection regardless of the YAML, because Splunk's defaults are
wrong for scheduled detections:

| Setting | Value | Why |
|---------|-------|-----|
| `alert.digest_mode` | `1` | Splunk defaults to `0`, which runs the alert action once **per result row** — twenty matched events would open twenty IRIS alerts. Digest mode sends one alert carrying all results. |
| `realtime_schedule` | `0` | Splunk defaults to `1`, which makes the scheduler **skip** runs it has fallen behind on instead of running them late, silently dropping coverage for those windows. `0` backfills. |
| `alert_type` | `number of results` | **Not** `number of events`. For a transforming search — which is most detections — Splunk's `number of events` counts what the base search matched *before* the transform. Measured in the lab, `EventCode=4625 \| stats count by user \| where count > 99999` gives eventCount=55 and resultCount=0, so a detection using `number of events` fires on every run while producing no results. Override per detection with `trigger.type`. |
| `alert.track` | `1` | The trigger appears in Splunk's Triggered Alerts. |
| `schedule_window` | `0` | Run on the scheduled minute, so the search window stays aligned with the cron interval. |

A consequence of digest mode: **one detection firing produces one IRIS alert,
however many events matched.** Write `alert.description` for that — describe the
pattern, not a single event. The matched rows are attached to the alert (capped
at 100).

## Deliberately not in this file

- `customer_id`, IRIS and Splunk URLs, credentials.
- Numeric severity ids.
- Splunk `action.*` settings. The pipeline attaches the IRIS action to every detection.

## Open questions

- **Environment scoping.** `customer_id` is one value per detection, but the
  `wineventlog` index holds hosts from both environments. A detection over it
  files every match under a single customer. The likely fix is scoping the SPL
  per environment, e.g. with a host-to-environment lookup and a macro.
- **One detection per technique ID.** Two detections for the same technique
  need an `id` suffix, since `id` is also the Splunk search name.
- **Tests.** Add a `tests` block (sample events + expected result) once a test
  approach is chosen.

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

## How fields map to what `send_to_iris.py` reads

The alert action looks up `metadata/<search_name>.yaml` at run time and reads
five keys. The pipeline generates that file from the detection YAML:

| Generated key | Source |
|---------------|--------|
| filename `<search_name>.yaml` | `id` |
| `alert_title` | `alert.title` |
| `alert_description` | `alert.description` |
| `severity_id` | `severity`: informational=2, low=3, medium=4, high=5, critical=6 |
| `customer_id` | `environment`: homelab=1, attack-range=2 (mapping kept in the private repo) |
| `tags` | `alert.tags` joined with `,` (no spaces) |

Splunk side, generated into `savedsearches.conf`: `search`, `cron_schedule`,
`dispatch.earliest_time` / `latest_time`, the alert trigger, suppression,
`disabled` (from `status`), and the IRIS alert action.

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

# CIM data models: required tags and core fields

The CIM add-on (`Splunk_SA_CIM`) ships the model JSON at
`$SPLUNK_HOME/etc/apps/Splunk_SA_CIM/default/data/models`. Always confirm exact required vs
recommended fields against the reference table for the **specific CIM add-on version installed** —
fields drift between major versions. Below are the high-traffic models and their anchor tags/fields.

## How membership works

An event joins a dataset when its eventtype carries the dataset's required **tags**. Fields are then
expected to be populated to CIM names. Tags are the gate; fields are the payload.

## Common models

### Authentication
- Tags: `authentication` (plus `default`, `insecure`, `privileged` on relevant sub-datasets).
- Core fields: `action` (success|failure|error), `app`, `src`, `dest`, `user`, `src_user`,
  `signature`.

### Network Traffic
- Tags: `network`, `communicate`.
- Core fields: `action` (allowed|blocked|teardown), `src`, `dest`, `src_port`, `dest_port`,
  `transport`, `bytes`, `bytes_in`, `bytes_out`, `protocol`.

### Web
- Tags: `web`.
- Core fields: `action`, `src`, `dest`, `url`, `http_method`, `status`, `http_user_agent`,
  `uri_path`, `bytes`.

### Endpoint (Processes / Filesystem / Registry / Ports / Services)
- Tags: per sub-dataset — e.g. `process` + `report`, `filesystem`, `registry`, `listening` + `port`.
- Core fields: `dest`, `user`, `process`, `process_name`, `process_id`, `parent_process`,
  `file_name`, `file_path`.

### Intrusion Detection
- Tags: `ids`, `attack`.
- Core fields: `action`, `src`, `dest`, `severity`, `signature`, `category`, `user`, `vendor_product`.

### Malware
- Tags: `malware`, `attack` (operations) / `malware`, `report` (attacks dataset varies by version).
- Core fields: `action`, `dest`, `file_name`, `file_hash`, `signature`, `vendor_product`.

### Change (Change Analysis)
- Tags: `change` (+ `audit`, `account`, `endpoint` on sub-datasets).
- Core fields: `action`, `dest`, `object`, `object_category`, `object_path`, `user`, `status`.

### Performance
- Tags: per sub-dataset — `performance` + one of `cpu`, `memory`, `storage`, `network`.
- Core fields: `dest`, plus metric fields like `cpu_load_percent`, `mem_used`, `storage_used_percent`.

### Data Access / Email / DLP / Vulnerabilities / Inventory
- Each has its own tag set; consult the version's reference table. The pattern is identical:
  tag → required fields.

## Reading the reference table

For any field you're unsure about:
- **Data type** (string vs number) matters — e.g. Performance wants numeric metrics; severity in
  some models wants a **string** (`critical`/`high`/…), so an integer `severity_id` must be mapped
  to a string `severity` via a lookup or EVAL.
- **Prescribed values** matter — `action`, `status`, `severity` have expected vocabularies. Map raw
  values into the CIM vocabulary with `EVAL`/`case()` or a lookup, don't pass raw codes through.

⚠️ **Best Practice — map to the *most specific* applicable model**, and only the fields that exist in
your data. Over-claiming membership (tagging events into a model whose fields you can't populate)
pollutes ES dashboards with empty rows.

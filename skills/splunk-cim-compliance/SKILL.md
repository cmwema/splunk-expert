---
name: splunk-cim-compliance
description: >-
  Normalize Splunk data to the Common Information Model (CIM) so it powers Enterprise Security, ITSI,
  and accelerated data models. Use this whenever the user is mapping a sourcetype to a CIM data model,
  normalizing field names, tagging events for a data model, or troubleshooting why data isn't showing
  up in ES / a data model — including phrasings like "make this CIM compliant", "map to the
  Authentication/Network Traffic/Web data model", "src/dest/user normalization", "FIELDALIAS",
  "eventtypes and tags for CIM", "events not appearing in datamodel", "CIM validator", or "tstats
  against an accelerated data model returns nothing". Covers the eventtype→tag→fieldalias→lookup
  workflow, the conf files involved (props/transforms/eventtypes/tags), per-model field/tag
  requirements, validation, and acceleration. Defer raw regex field-extraction mechanics and full TA
  packaging to the TA-development skill; this skill is specifically about CIM normalization.
---

# Splunk CIM compliance

The Splunk Common Information Model (CIM) is a set of pre-defined data models — each a least-common-
denominator schema of field names and tags for one domain (Authentication, Network Traffic, Web,
Endpoint, Intrusion Detection, Performance, etc.). Normalize a source to CIM once and every CIM-aware
search, dashboard, and correlation works against it without per-source rewrites.

⚠️ **Why it matters.** Enterprise Security is entirely CIM-dependent — non-compliant data is
invisible to ES. ITSI KPIs, accelerated `tstats`, and Pivot all rely on the same models. CIM is the
contract between your raw data and everything downstream.

## The normalization workflow (always this order)

1. **Identify the target model + dataset.** Find the CIM data model whose domain matches the events
   (e.g. login events → Authentication). Open its reference table for the required tags and fields.
2. **Create event types** (`eventtypes.conf`) that select the events.
3. **Tag the event types** (`tags.conf`) with the tags the dataset requires.
4. **Alias / extract fields** (`props.conf` FIELDALIAS, EVAL, REPORT, LOOKUP) to the CIM field names.
5. **Validate** against the model (field picker, Pivot/Datasets, `datamodel`/`tstats`).
6. **Accelerate** the model if production dashboards/correlations need it.

## Routing table

| If the user is… | Read |
|---|---|
| choosing a model and needs the required fields/tags | `references/models-and-fields.md` |
| writing the eventtypes/tags/props config | `references/normalization-config.md` |
| validating or debugging "data not in the model" | `references/validation-and-accel.md` |

## Shared conventions

- **Search-time, not index-time.** CIM normalization is search-time (FIELDALIAS/EVAL/LOOKUP/tags).
  Do not bake CIM fields in at index time — it's wasteful and inflexible.
- **Keep normalization in a dedicated add-on.** Put props/transforms/eventtypes/tags in a `TA-` (or
  `SA-`/`DA-`) add-on, not in app logic, so it deploys to search heads independently and is reusable.
- **Don't rename — alias.** Preserve the original field; add a CIM field alongside it
  (`FIELDALIAS-x = orig AS cim_name`). Downstream that needs the raw value still has it.
- **Tags drive model membership.** An event belongs to a dataset because its eventtype carries the
  right tags — not because of field names. Get the tags right first.
- **`src`/`dest`/`user`/`action` are the workhorses.** Most models key on these; normalize them
  first, then fill model-specific fields.
- **Set sharing in `metadata/default.meta`** so eventtypes/tags/aliases are global, otherwise the
  model only sees them in one app context.

## Quick example (Authentication)

```ini
# eventtypes.conf
[acme_gateway_auth]
search = sourcetype=acme:gateway:auth

# tags.conf
[eventtype=acme_gateway_auth]
authentication = enabled

# props.conf
[acme:gateway:auth]
FIELDALIAS-user = account_name AS user
FIELDALIAS-src  = client_ip    AS src
FIELDALIAS-dest = gateway_host AS dest
EVAL-action = case(status="OK","success",status="FAIL","failure",1=1,"unknown")
```

Then validate: `| datamodel Authentication Authentication search | search sourcetype=acme:gateway:auth`
should return your events with `user`, `src`, `dest`, `action` populated.

## Next steps

1. Run the CIM Validator (custom command shipped with the CIM add-on) on the source.
2. Confirm sharing/permissions are global.
3. Only then enable acceleration (see `references/validation-and-accel.md`).

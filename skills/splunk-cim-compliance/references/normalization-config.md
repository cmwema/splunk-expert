# Writing the normalization config

All four conf files live in a dedicated add-on (`TA-<vendor>-<product>`), under `default/`. Set
sharing global in `metadata/default.meta`.

## 1. eventtypes.conf — select the events

```ini
[acme_gateway_auth]
search = sourcetype=acme:gateway:auth

# Conditional eventtype (only failures, or severity threshold):
[ossec_attack]
search = sourcetype=ossec severity_id>=6
```

One eventtype per logical event class. Keep the search cheap and specific (anchor on `sourcetype`).

## 2. tags.conf — declare model membership

```ini
[eventtype=acme_gateway_auth]
authentication = enabled

[eventtype=ossec_attack]
ids = enabled
attack = enabled
```

The keys are the model's required tags; value is always `enabled`. This is what actually puts events
in the dataset.

## 3. props.conf — normalize fields

Field aliases (preserve original, add CIM name):

```ini
[acme:gateway:auth]
FIELDALIAS-user = account_name AS user
FIELDALIAS-src  = client_ip    AS src
FIELDALIAS-dest = gateway_host AS dest
```

Multiple aliases can live in one stanza. **FIELDALIAS does not support multivalue fields** — use an
EVAL for those.

Calculated/normalized values (map raw codes into CIM vocabulary):

```ini
EVAL-action = case(status_code=="200","success", status_code=="403","failure", 1=1,"unknown")
EVAL-severity = case(severity_id>=9,"critical", severity_id>=6,"high", 1=1,"low")
```

Lookups (e.g. integer severity → string, or product/vendor enrichment):

```ini
LOOKUP-severity = ossec_severities_lookup severity_id OUTPUT severity
```

Static vendor/product via a REPORT or EVAL:

```ini
EVAL-vendor = "Acme"
EVAL-product = "Notifications Gateway"
EVAL-vendor_product = "acme_notifications_gateway"
```

## 4. transforms.conf — when you need extractions or static lookups

For field extractions feeding the aliases (regex/delimited), or static lookups referenced by
`LOOKUP-`:

```ini
[ossec_severities_lookup]
filename = ossec_severities.csv
```

## metadata/default.meta — make it global

```ini
[]
access = read : [ * ], write : [ admin ]
export = system
```

⚠️ **Best Practice — order of operations within search-time.** Splunk applies extractions →
field aliases → calculated (EVAL) fields → lookups. So an `EVAL-action` can reference an aliased
field, and a `LOOKUP-` can reference an EVAL'd key, but not vice versa. If a normalized field is
empty, check you're not referencing something computed later in the chain.

⚠️ **Best Practice — never hardcode credentials** in transforms that call external lookups/commands;
use `passwords.conf` via the Splunk secret store.

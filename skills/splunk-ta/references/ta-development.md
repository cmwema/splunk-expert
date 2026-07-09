# TA / App Development — CIM, AppInspect, Packaging

Reference for building Splunk Technology Add-ons and apps. Default to a clean separation: knowledge
objects (props/transforms/eventtypes/tags) in a TA, app logic separate.

## Contents
- App vs add-on, naming, and separation
- Directory structure
- CIM compliance workflow
- Custom REST handlers (non-EAI persistent pattern)
- app.conf and metadata
- AppInspect and local validation
- Packaging

## App vs add-on, naming, and separation

- **TA-** Technology Add-on: inputs + parsing + CIM mapping for a data source. `TA-<vendor>-<product>`.
- **SA-** Supporting Add-on: shared logic/lookups used by multiple apps.
- **DA-** Domain Add-on: dashboards/views for a security or IT domain.

Keep knowledge objects (props.conf, transforms.conf, eventtypes.conf, tags.conf) in the TA, away
from app UI logic, so a TA can be deployed to indexers/HFs without dragging UI along.

## Directory structure

Use the bundled `assets/ta-skeleton/`. Canonical layout:

```
TA-vendor-product/
├── default/
│   ├── app.conf
│   ├── props.conf
│   ├── transforms.conf
│   ├── eventtypes.conf
│   ├── tags.conf
│   └── inputs.conf
├── metadata/
│   └── default.meta
├── bin/                 # scripted/modular inputs, REST handlers
├── lookups/
├── static/              # app icons
└── README.md
```

## CIM compliance workflow

CIM normalization lets accelerated data models and ES/ITSI content work against your data.

1. **Pick the data model.** Map the sourcetype to the most specific applicable model
   (Authentication, Network_Traffic, Web, Malware, Change, etc.).
2. **Normalize fields.** Alias or eval source fields to CIM names:
   ```ini
   [acme:fw]
   FIELDALIAS-src   = source_address AS src
   FIELDALIAS-dest  = dest_address   AS dest
   EVAL-action      = case(verdict=="permit","allowed", verdict=="deny","blocked", true(),"unknown")
   ```
3. **Tag the events** so the data model recognizes them — `eventtypes.conf` + `tags.conf`:
   ```ini
   # eventtypes.conf
   [acme_fw_traffic]
   search = sourcetype=acme:fw
   ```
   ```ini
   # tags.conf
   [eventtype=acme_fw_traffic]
   network = enabled
   communicate = enabled
   ```
4. **Validate** against the CIM data model required/expected fields (use the CIM Validator app) before
   shipping.

⚠️ **Best Practice** — Normalize at search time via FIELDALIAS/EVAL; avoid index-time CIM fields.
Index-time changes are irreversible without re-indexing and bloat the index.

## Custom REST handlers (non-EAI persistent pattern)

For endpoints that aren't backed by a conf file (non-EAI), use the persistent handler pattern.
Register in `restmap.conf`, optionally expose through `web.conf`, and subclass
`PersistentServerConnectionApplication`.

```ini
# default/restmap.conf
[script:my_handler]
match = /my_endpoint
scripttype = persist
handler = my_module.MyHandler
output_modes = json
passSystemAuth = true
```

```python
# bin/my_module.py
import json, logging
from logging.handlers import RotatingFileHandler
import os

def setup_logger(level):
    logger = logging.getLogger("my_handler")
    logger.setLevel(level)
    logger.propagate = False
    if not logger.handlers:
        h = RotatingFileHandler(
            os.path.join(os.environ["SPLUNK_HOME"], "var", "log", "splunk", "my_handler.log"),
            maxBytes=10_000_000, backupCount=5,
        )
        h.setFormatter(logging.Formatter("%(asctime)s %(levelname)s %(message)s"))
        logger.addHandler(h)
    return logger

logger = setup_logger(logging.INFO)

from splunk.persistconn.application import PersistentServerConnectionApplication

class MyHandler(PersistentServerConnectionApplication):
    def __init__(self, command_line=None, command_arg=None):
        super().__init__()

    def handle(self, in_string):
        try:
            req = json.loads(in_string)
            authtoken = req.get("session", {}).get("authtoken")
            method = req.get("method")
            query = dict(req.get("query", []))
            # ... guard query params against SPL injection before using them ...
            payload = {"ok": True, "method": method}
            return {"payload": json.dumps(payload), "status": 200}
        except Exception as e:
            logger.error("handler error: %s", e)
            return {"payload": json.dumps({"error": str(e)}), "status": 500}
```

⚠️ **Best Practice** — Always return a `{'payload', 'status'}` dict on *every* path including errors;
guard any `query` params that feed SPL against injection; never `verify=False` on outbound calls;
gate verbose logging behind a flag; `logger.propagate = False` to avoid duplicate lines in splunkd.

## app.conf and metadata

```ini
# default/app.conf
[install]
is_configured = 0

[ui]
is_visible = 1
label = Acme Firewall TA

[launcher]
author = Your Team
description = Inputs and CIM mapping for Acme Firewall
version = 1.0.0

[package]
id = TA-acme-firewall
```

```ini
# metadata/default.meta
[]
access = read : [ * ], write : [ admin ]
export = system
```

## AppInspect and local validation

Run `scripts/validate_ta.py <path-to-TA>` (bundled) for fast local checks: app.conf metadata
presence, `default.meta` existence, conf-file parse sanity, and common pitfalls. It is a pre-flight,
not a replacement for Splunk AppInspect, which you should run before Splunkbase or internal
distribution.

⚠️ **Best Practice** — Run AppInspect (CLI or API) on every release. Fix `failure`/`error` items;
`warning`s are usually worth addressing too.

## Packaging

A TA is a tarball of the app directory, with the top-level directory name matching `[package] id`
in `app.conf`. For the full build process — what to exclude, avoiding a self-included/nested
archive, version-bump conventions, and verifying the result before shipping — see
`splunk-app-packaging`.

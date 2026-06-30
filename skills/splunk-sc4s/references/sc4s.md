# Splunk Connect for Syslog (SC4S)

Reference for SC4S — a containerized syslog-ng instance that parses, filters, routes, and forwards
syslog to Splunk via HEC. Pin a stable release in production; never use `latest`.

Pipeline to keep in mind always:
`Source (device) → SC4S (parse / filter / route) → HEC → Splunk Indexer → Index`

## Contents
- Architecture and deployment
- env_file configuration
- Parser development (custom block + application block)
- Filtering and routing
- splunk_index.csv
- Timestamp handling
- Testing with sc4s-test
- Tuning and monitoring
- Best practices

## Architecture and deployment

SC4S runs under Docker or Podman. All customization lives under `/opt/sc4s/local/` so it survives
container upgrades — never modify files under `/opt/sc4s/` directly.

Directory layout:
- `/opt/sc4s/local/config/app-parsers/` — custom parsers and routing overrides
- `/opt/sc4s/local/context/splunk_index.csv` — index routing lookup
- `/opt/sc4s/env_file` — environment variables
- `/opt/sc4s/log/` — runtime logs for troubleshooting

Pin the image: `ghcr.io/splunk/splunk-connect-for-syslog:<version>`.

## env_file configuration

```bash
# --- HEC destination ---
SC4S_DEST_SPLUNK_HEC_DEFAULT_URL=https://hec.splunk.example.com:8088
SC4S_DEST_SPLUNK_HEC_DEFAULT_TOKEN=<hec-token>
SC4S_DEST_SPLUNK_HEC_DEFAULT_TLS_VERIFY=yes

# --- Default index (override per-source via splunk_index.csv) ---
SC4S_DEST_SPLUNK_HEC_DEFAULT_INDEX=netsyslog

# --- Disk buffering: prevents data loss when HEC is unavailable ---
SC4S_DEST_SPLUNK_HEC_DEFAULT_DISKBUFF_ENABLE=yes
SC4S_DEST_SPLUNK_HEC_DEFAULT_DISKBUFF_PATH=/opt/sc4s/local/buffer

# --- Listeners (enable only what you use) ---
SC4S_LISTEN_DEFAULT_TCP_PORT=514
SC4S_LISTEN_DEFAULT_UDP_PORT=514
SC4S_LISTEN_DEFAULT_TLS_PORT=6514

# --- Performance / logging ---
SC4S_SOURCE_TLS_ENABLE=yes
SC4S_LOG_LEVEL=info
```

⚠️ **Best Practice** — Enable disk buffering (`...DISKBUFF_ENABLE=yes`). Without it, an HEC outage
drops events silently. Set `TLS_VERIFY=yes` — disabling cert verification is a production no-go.

## Parser development

When no built-in parser handles a source, create
`/opt/sc4s/local/config/app-parsers/<vendor>_<product>.conf`. Two blocks: a `block parser` that
extracts fields and sets metadata, and an `application` that matches and invokes the parser.

```syslog-ng
block parser acme_fw-parser() {
    channel {
        rewrite {
            set("acme_fw"   value("fields.sc4s_vendor_product"));
            set("acme"      value("fields.sc4s_vendor"));
            set("fw"        value("fields.sc4s_product"));
        };
        parser {
            # choose the parser that fits the format:
            #   kv-parser()  csv-parser()  json-parser()  regexp-parser()
            kv-parser(prefix(".values."));
        };
        if {
            # success path: assign final sourcetype (and index if overriding)
            rewrite {
                set("acme:fw" value(".splunk.sourcetype"));
            };
        } elif {
            # fallback so nothing silently lands in a catch-all
            rewrite { set("acme:fw:fallback" value(".splunk.sourcetype")); };
        };
    };
};

application acme_fw[sc4s-network-source] {
    filter {
        host("fw-.*\.acme\.local" type(pcre))
        or netmask(10.20.0.0/16)
        or message("ACME-FW" type(string));
    };
    parser { acme_fw-parser(); };
};
```

Metadata you set in the parser:
- `.splunk.sourcetype` — Splunk sourcetype (required)
- `.splunk.index` — override target index (optional; otherwise default/`splunk_index.csv`)
- `fields.sc4s_vendor`, `fields.sc4s_product`, `fields.sc4s_vendor_product`

`assets/sc4s-parser.conf.tmpl` (in this skill) is a fill-in-the-blanks version of the above.

## Filtering and routing

Filter options inside the `application` block:
- `host("<regex>" type(pcre))` — match hostname/IP by regex
- `netmask(<CIDR>)` / `netmask6(...)` — match source IP range
- `program("<name>")` — match syslog program/tag
- `message("<pattern>" type(pcre|string))` — match raw message content
- `tags("...")` — match SC4S internal tags

Combine selective filters; lead with the cheapest (netmask/host) before regex on message.

**Drop filter** for noisy sources (apply early to cut HEC load):
```syslog-ng
application drop_acme_noise[sc4s-postfilter] {
    filter { message("health-check probe" type(string)); };
    flags(final);
    rewrite { set("0" value("fields.sc4s_ok")); };
};
```

## splunk_index.csv

Routes a `vendor_product` to a Splunk index without editing parsers
(`/opt/sc4s/local/context/splunk_index.csv`):

```csv
vendor_product,index,source,sourcetype
acme_fw,network,,
cisco_asa,network,,
linux_secure,linux,,
```

## Timestamp handling

```syslog-ng
parser {
    date-parser(
        format("%b %d %H:%M:%S")
        flags(guess-timezone)        # use when the message carries no TZ
        template("${MESSAGE}")
    );
};
```

Validate parsed time against sample messages; RFC 3164 lacks year and timezone, so `guess-timezone`
and a sensible default matter. RFC 5424 carries a full timestamp — prefer it when the device supports it.

## Testing with sc4s-test

Write unit tests before deploying a parser. `sc4s-test` feeds a known message and asserts on the
resulting sourcetype, index, and extracted fields. Keep a test per parser; run them in CI.

Also check runtime health:
- `/opt/sc4s/log/` for parser errors
- HEC acknowledgement issues (token, URL, TLS, index existence on the indexer)

## Tuning and monitoring

- `SC4S_DEST_SPLUNK_HEC_DEFAULT_WORKERS` — HEC worker count; raise for high volume.
- Disk buffer sizing/location — put the buffer on fast local disk with headroom.
- Worker threads and batch size — tune to event rate.
- Monitor in the SC4S internal metrics: `sc4s_events_per_second` (throughput) and
  `sc4s_dest_failed` (delivery failures). Rising `sc4s_dest_failed` means HEC trouble.

## Best practices

⚠️ **Best Practice** summary:
- TCP+TLS transport when the device supports it; UDP only as a fallback.
- Never use the `default` catch-all sourcetype in production — build explicit parsers.
- Drop filters early to reduce HEC load from noisy devices.
- Keep all customization in `/opt/sc4s/local/`; pin the container version.
- Test every parser with `sc4s-test` before production.
- Pair each SC4S parser with a corresponding Splunk TA for CIM mapping at search time.
- Keep the `sc4s_vendor_product` identifier consistent across parser, `splunk_index.csv`, and TA.

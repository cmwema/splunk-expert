# Index design and naming

## Separate by data domain

⚠️ **Best Practice — one index per domain, not one mega-index and not one-per-source.** Split for
three reasons: **RBAC** (grant teams only their indexes), **retention** (different data, different
lifetimes), **search efficiency** (searches scan fewer buckets).

Typical domain split:

```
index=windows     Windows event logs, Sysmon
index=linux       *nix syslog/auth/audit/journald
index=network     firewall, switch, LB, DNS, NetFlow
index=syslog      catch-all for SC4S sources without a dedicated home (still explicit sourcetypes)
index=app_<name>  per major application
index=_audit      (built-in) Splunk audit — monitor it
```

Avoid the extremes: a single `main` for everything (no RBAC, slow), or an index per host/source
(bucket explosion, management overhead).

## Naming conventions

- lowercase, domain-first (`network_firewall` or just `network` if firewall shares with other net
  gear), no spaces.
- Keep names stable — renaming an index is disruptive.
- Mirror naming across `indexes.conf`, SC4S `splunk_index.csv`, and dashboards/SPL so routing is
  predictable.

## Defining indexes (clustered vs standalone)

⚠️ **Best Practice — define indexes on the Cluster Manager** for clustered indexers, in the CM's
`indexes.conf`, then push the bundle (`splunk apply cluster-bundle`). A misconfigured index on the CM
affects all peers — validate before pushing. For standalone indexers, define in
`$SPLUNK_HOME/etc/apps/<app>/local/indexes.conf`.

```ini
[network]
homePath   = volume:hot/$_index_name/db
coldPath   = volume:cold/$_index_name/colddb
thawedPath = $SPLUNK_HOME/var/lib/splunk/$_index_name/thaweddb
maxDataSize = auto_high_volume
frozenTimePeriodInSecs = 7776000        # 90 days
```

(Use `volume:` references so tier sizing is centralized — see retention-and-sizing.md.)

## Routing data to the right index

- **UF/HF**: set `index=` per `inputs.conf` stanza, or override on an HF via `transforms.conf`
  routing.
- **SC4S**: set `.splunk.index` in the parser, or map `vendor_product → index` in `splunk_index.csv`.
- **HEC**: set the index on the token (and allow only the indexes it should write to).

## Access control

Grant index access per role (least privilege). Index-level controls prevent cross-team data exposure
— a network-ops role shouldn't search HR app indexes. Pair with knowledge-object sharing so a team's
TAs/searches are visible only where appropriate.

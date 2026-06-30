# Splunk SOAR — Playbooks and Custom Python Actions

Reference for SOAR automation: playbook design, custom Python functions, and integration patterns.
Python 3.x.

## Contents
- Playbook design principles
- Custom function code pattern
- Decision blocks and error handling
- Assets, credentials, and parameterization
- Best practices

## Playbook design principles

- **Idempotency first.** Every action must tolerate re-runs without duplicate side effects (don't
  create a second ticket, don't re-block an already-blocked IP). Check current state before acting.
- Use **decision blocks** to branch on action success/failure explicitly rather than assuming success.
- Keep playbooks modular — a parent playbook calling focused child playbooks is easier to test than
  one monolith.

## Custom function code pattern

```python
import phantom.rules as phantom
import json

def block_ip(container=None, **kwargs):
    """Block a source IP on the firewall, idempotently."""
    try:
        # Pull artifacts; validate input exists before acting
        ips = phantom.collect2(container=container, datapath=['artifact:*.cef.sourceAddress'])
        if not ips:
            phantom.debug("no source IP found; nothing to block")
            return

        for (ip,) in ips:
            # idempotency: skip if already blocked (check a custom list or prior action)
            already = phantom.get_list(list_name="blocked_ips")
            if already and any(ip in row for row in already[2]):
                phantom.debug(f"{ip} already blocked; skipping")
                continue

            phantom.act(
                action="block ip",
                parameters=[{"ip": ip}],
                assets=["firewall_asset"],
                callback=block_ip_callback,
            )
    except Exception as e:
        phantom.error(f"block_ip failed: {e}")
        raise

def block_ip_callback(action, success, container, results, handle, **kwargs):
    if not success:
        phantom.error("block ip action failed")
        return
    phantom.debug("block ip succeeded")
```

⚠️ **Best Practice** — Wrap custom Python in `try/except`, log meaningful errors with
`phantom.error()`, and use `phantom.debug()` during development (gate verbose debug behind a flag for
production). Use callbacks to confirm action success instead of assuming it.

## Decision blocks and error handling

- After every external action, branch on success. A failed enrichment shouldn't silently proceed to
  a containment step.
- Set sane timeouts and handle partial results (some IOCs enrich, some don't).

## Assets, credentials, and parameterization

- Parameterize endpoints, credentials, and thresholds via **asset configurations** and **custom
  lists** — never hardcode them in playbook code.
- Apply **least privilege** to each asset: it should hold only the permissions its specific actions
  need (a firewall-block asset doesn't need admin on the SIEM).
- For outbound REST from custom code, validate TLS — never `verify=False` in production.

## Best practices

⚠️ **Best Practice** summary:
- Idempotent playbooks; check state before side-effectful actions.
- `try/except` + `phantom.error()` in every custom function; `phantom.debug()` gated for prod.
- Decision blocks for explicit failure handling.
- Parameterize via assets/custom lists; least-privilege assets.
- Validate TLS on outbound calls.

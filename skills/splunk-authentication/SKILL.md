---
name: splunk-authentication
description: >-
  Splunk authentication and authorization — all tasks. Trigger for: LDAP/Active
  Directory auth, SAML SSO, authentication.conf, authorize.conf, user roles,
  capabilities, role inheritance, LDAP group-to-role mapping, SAML IdP config
  (Okta, Azure AD, ADFS, Ping), SAML attribute mapping, multi-factor auth (MFA),
  Splunk native auth, RADIUS auth, password policy, token-based auth, user
  management, index-level access control, search filter restrictions,
  audit.conf, login auditing, proxy auth, SPL injection via role filter.
  Default: Splunk Enterprise 9.x.
---

# Splunk Authentication & Authorization

Router only. Read `references/authentication.md` for all auth tasks.

## Routing

| Task | Action |
|---|---|
| LDAP / Active Directory integration | Read `references/authentication.md` |
| SAML SSO configuration (any IdP) | Read `references/authentication.md` |
| Roles, capabilities, index access | Read `references/authentication.md` |
| Token auth, password policy | Read `references/authentication.md` |
| Audit logins, failed auth searches | Read `references/authentication.md` |

## Shared conventions

- LDAP: bind with a dedicated read-only service account — never the domain admin.
- Always map LDAP groups to Splunk roles; avoid assigning roles directly to individual users.
- SAML: set `attributeQuerySoapPassword` and `signedAssertion = true`; validate cert chains.
- Capabilities follow least-privilege: grant `search` + specific index access; avoid `admin_all_objects`.
- `authorize.conf` role `srchIndexesAllowed` and `srchIndexesDefault` define index scope — always set both.
- Use `srchFilter` in `authorize.conf` to restrict data visible to a role at the SPL level (e.g. by `host`, `sourcetype`).
- Token auth: set short `expiration` for programmatic tokens; rotate via REST (`/services/authorization/tokens`).
- Audit auth events: `index=_audit action=login` — alert on repeated failures or off-hours logins.
- Never disable SSL on the management port (`splunkd`) — all auth flows over 8089 must be encrypted.

## Output format

Fenced `ini` for conf files, `xml` for SAML metadata snippets. Section headers: authentication.conf / authorize.conf / LDAP Map / SAML IdP / Role Design. ⚠️ callouts for privilege escalation and SAML validation issues.

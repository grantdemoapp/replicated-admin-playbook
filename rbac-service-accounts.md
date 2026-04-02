# RBAC & Service Accounts

## Creating a Policy

```bash
curl -X POST \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  "https://api.replicated.com/vendor/v3/policy" \
  -d '{
    "name": "Developer",
    "description": "Full access except Stable promotion",
    "definition": "{\"v1\":{\"name\":\"Developer\",\"resources\":{\"allowed\":[\"**/*\"],\"denied\":[\"kots/app/*/channel/<STABLE_ID>/promote\"]}}}"
  }'
```

The definition must be a JSON-encoded string (string-within-JSON). Use pretty-printed JSON inside if you want it readable in the vendor portal.

## Creating a Service Account

```bash
# Note: use "policy_id" (singular) — "policy_ids" is silently ignored on create
curl -X POST \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  "https://api.replicated.com/vendor/v3/team/serviceaccount" \
  -d '{"name":"github-actions-ci","policy_id":"<policy-id>"}' \
  | jq '{name: .service_account.name, token: .access_token}'
```

Store the `access_token` — it's only shown once.

## Listing Service Accounts

```bash
curl -H "Authorization: $TOKEN" \
  "https://api.replicated.com/vendor/v3/team/serviceaccounts"
```

## The AI Agent Policy

Recommended deny list for an automated agent token:

```
kots/app/*/channel/*/promote          # no direct channel promotion
platform/app/*/channel/*/promote
kots/app/*/installer/promote
kots/app/*/channel/create             # no channel structure changes
kots/app/*/channel/*/delete
kots/app/*/channel/*/archive
kots/app/*/channel/*/update
kots/app/*/delete                     # no deleting apps
kots/app/*/update                     # no app settings changes
kots/externalregistry/create          # no registry mutations
kots/externalregistry/*/delete
kots/externalregistry/update
kots/app/*/customhostname/create      # no custom domain changes
kots/app/*/customhostname/delete
kots/app/*/customhostname/default/set
kots/app/*/customhostname/default/unset
kots/app/*/license/*/delete           # no deleting licenses
kots/license/*/delete
kots/app/*/licensefields/create       # no license field schema changes
kots/app/*/licensefields/update
kots/app/*/licensefields/delete
kots/app/*/builtin-licensefields/update
kots/app/*/builtin-licensefields/delete
team/member/invite                    # no team membership changes
team/members/create
team/members/delete
team/policy/create                    # no RBAC changes
team/policy/update
team/policy/delete
team/authentication/update            # no auth config (SAML, Google, auto-join)
team/security/update                  # no MFA/password policy
```

## RBAC Limitation: No License Type Restriction

The `kots/app/*/license/create` resource is binary — you cannot restrict it to specific license types (dev-only, etc.) through RBAC alone.

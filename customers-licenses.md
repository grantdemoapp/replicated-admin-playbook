# Customers & Licenses

## Creating Customers

```bash
replicated customer create \
  --app my-app \
  --name "Acme Corp" \
  --channel Stable \
  --expires-in 8760h \
  --type dev   # dev | trial | paid | community | test
```

Types: `dev` and `test` licenses don't expire by default and are for internal use. `trial` and `paid` are for real customers.

## Downloading a License

```bash
replicated customer download-license \
  --app my-app \
  --customer "Acme Corp" \
  > /tmp/license-acme.yaml
```

Note: The `--customer` flag takes the customer **name**, not ID. For helm-only customers (KOTS disabled), the standard `download-license` may return a 403 — use the API endpoint instead:
```bash
curl -H "Authorization: $TOKEN" \
  "https://api.replicated.com/vendor/v3/app/<appId>/customer/<customerId>/helm-license-download"
```

## License Fields (Custom Entitlements)

Create a custom field via API (note: it's `/license-field` singular, not `/licensefields`):
```bash
curl -X POST \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  "https://api.replicated.com/vendor/v3/app/<appId>/license-field" \
  -d '{"name":"seats","title":"Seats","type":"Integer","default":"5"}'
```

List fields:
```bash
curl -H "Authorization: $TOKEN" \
  "https://api.replicated.com/vendor/v3/app/<appId>/license-fields"
```

## The `expires_at` Field

The `expires_at` field is a built-in entitlement — not a top-level field. To check expiry via the SDK:

```typescript
// Wrong - expiresAt doesn't exist at top level
const expired = new Date(licenseInfo.expiresAt) < new Date();

// Correct - it's in entitlements
const expiresAt = licenseInfo?.entitlements?.expires_at?.value;
const expired = expiresAt && expiresAt !== "" && new Date(expiresAt) < new Date();
```

Blank string `""` and `"0001-01-01T00:00:00Z"` both mean "never expires".

## Channel Binding

The SDK binds to its channel at install time from the license. If an instance reports to the wrong channel or shows empty `versionLabel`:

1. The license was from a different channel than expected
2. The only fix is reinstall with the correct license — there is no post-install patch

## Per-Customer Entitlement Overrides

To set a custom value for a specific customer, use the vendor portal UI (Customers → Edit → License Fields). The REST API endpoint for per-customer entitlement overrides is not publicly documented and has been inconsistently available.

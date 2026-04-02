# SDK Integration

## Correct API Endpoints

All requests go to `http://replicated:3000` (in-cluster) or the value of `REPLICATED_SDK_URL`.

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/v1/app/info` | GET | Instance details, current release |
| `/api/v1/app/updates` | GET | Available upgrades |
| `/api/v1/app/history` | GET | Previously installed releases |
| `/api/v1/app/status` | GET | App readiness state |
| `/api/v1/app/custom-metrics` | PATCH | Upsert custom metrics |
| `/api/v1/app/instance-tags` | POST | Set instance tags |
| `/api/v1/license/info` | GET | License details + entitlements |
| `/api/v1/license/fields` | GET | All license fields with signatures |
| `/api/v1/license/fields/{name}` | GET | Single license field |

## `/app/info` Response Shape

`versionLabel` is **nested** under `currentRelease`, not at the top level:

```json
{
  "instanceID": "...",
  "appSlug": "my-app",
  "appName": "My App",
  "appStatus": "ready",
  "currentRelease": {
    "versionLabel": "0.1.72",   // ← HERE, not top-level
    "channelID": "...",
    "channelName": "Stable",
    "helmReleaseName": "my-chart",
    "helmReleaseRevision": 5,
    "helmReleaseNamespace": "default"
  },
  "channelID": "...",
  "channelName": "Stable",
  "channelSequence": 4,
  "releaseSequence": 30
}
```

## `/license/info` Response Shape

`expires_at` is **inside entitlements**, not a top-level field:

```json
{
  "licenseID": "...",
  "customerName": "Acme Corp",
  "licenseType": "dev",
  "entitlements": {
    "expires_at": {
      "title": "Expiration",
      "value": "2024-12-31T00:00:00Z",  // "" = never expires
      "valueType": "String"
    },
    "seats": {
      "title": "Seats",
      "value": 10,         // Integer fields come back as actual numbers
      "valueType": "Integer"
    }
  }
}
```

## Checking Expiry Correctly

```typescript
function isLicenseExpired(licenseInfo: LicenseInfo | null): boolean {
  const expiresAt = licenseInfo?.entitlements?.expires_at?.value;
  if (!expiresAt || expiresAt === "" || expiresAt === "0001-01-01T00:00:00Z") return false;
  return new Date(expiresAt as string) < new Date();
}
```

## Reading Custom License Fields

```typescript
// /license/fields returns a map keyed by field name
const fields = await fetch('/api/v1/license/fields').then(r => r.json());

// Integer fields come back as numbers, not strings
const seats = fields.seats?.value;  // number | string
const n = typeof seats === "number" ? seats : parseInt(seats, 10);
```

## Caching

All SDK fetches should use `cache: "no-store"` in Next.js server components. Using `revalidate` causes the update banner to show stale data (the old "update available" state persists for the revalidation window even after the instance upgrades).

## Instance Tags

```typescript
await fetch('http://replicated:3000/api/v1/app/instance-tags', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    data: {
      force: false,    // true = overwrite existing tags
      tags: {
        name: "my-instance-name",   // sets the display name in vendor portal
        app_type: "my-app-type",
      }
    }
  })
});
```

## Custom Metrics

```typescript
await fetch('http://replicated:3000/api/v1/app/custom-metrics', {
  method: 'PATCH',   // PATCH = upsert
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ data: { numProjects: 20 } })
});
```

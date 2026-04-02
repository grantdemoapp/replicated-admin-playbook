# Instance Tracking

## How Instances Are Counted

The Replicated SDK generates an instance ID from the cluster UID + namespace. Each unique cluster/namespace combination = one instance.

Consequences:
- `helm upgrade` on the same cluster = same instance, version updates in place ✓
- `helm uninstall && helm install` on the same cluster = same instance ✓  
- New cluster = new instance, even if using the same license ✗ (creates a new record)
- CI smoke test cluster = new instance every run ✗

## Checking Instance State via API

```bash
curl -H "Authorization: $TOKEN" \
  "https://api.replicated.com/vendor/v3/app/<appId>/customer/<customerId>/instances" \
  | jq '.instances[] | {id: .instanceId, version: .versionHistory[0].versionLabel, last: .lastActive}'
```

Note: `customerId` is the customer object ID (from `replicated customer ls --output json`), not the `licenseID`.

## Why `versionLabel` is Empty

If an instance shows `versionLabel: ""`:
1. It was installed with a license from the wrong channel (channel ID mismatch)
2. The SDK is pointing at the wrong update endpoint
3. The only fix is reinstall with the correct license

## Stale Instance Cleanup

Old/dead instances (e.g. from deleted CI clusters) persist in the vendor portal. They can be archived via the UI. They don't affect billing or license usage — they're just historical records.

## Using a Dedicated CI License

To prevent CI smoke tests from polluting real customer instance lists, create a dedicated test customer:

```bash
replicated customer create \
  --app my-app \
  --name "CI Test Customer" \
  --channel Unstable \
  --type test
```

Store the license as a GitHub secret (`TEST_LICENSE`) and use only for CI, never for real customer clusters.

## Instance Tags

Set meaningful tags to make instances identifiable in the vendor portal:

```typescript
setInstanceTags({
  name: "production-acme",      // shows as instance name in vendor portal
  environment: "production",
  app_type: "communications-portal",
});
```

The `name` tag sets the display name in the vendor portal instance list.

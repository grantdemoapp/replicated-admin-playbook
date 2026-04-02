# Channels

## Channel Strategy

Standard three-channel setup:
- **Unstable** — CI promotes here automatically on every merge to main
- **Beta** — optional; manual promotion after Unstable validation
- **Stable** — only humans (or a dedicated elevated token) can promote here

## Enforcing the Stable Gate

Use RBAC to prevent direct Stable promotion from CI tokens. Create a Developer policy:

```json
{
  "v1": {
    "name": "Developer",
    "resources": {
      "allowed": ["**/*"],
      "denied": [
        "kots/app/*/channel/<STABLE_CHANNEL_ID>/promote",
        "platform/app/*/channel/<STABLE_CHANNEL_ID>/promote"
      ]
    }
  }
}
```

Get the channel ID:
```bash
replicated channel ls --app my-app --output json | jq '.[] | select(.name=="Stable") | .id'
```

Create the policy via API:
```bash
curl -X POST \
  -H "Authorization: $REPLICATED_API_TOKEN" \
  -H "Content-Type: application/json" \
  "https://api.replicated.com/vendor/v3/policy" \
  -d '{"name":"Developer","description":"No direct Stable promotion","definition":"{\"v1\":{...}}"}'
```

## Promoting to Stable

```bash
# Get the sequence you want to promote
replicated release ls --app my-app

# Promote
replicated release promote <sequence> Stable --app my-app --version X.Y.Z
```

## Channel Sequence vs Release Sequence

- **Release sequence**: global, monotonically increasing across all channels
- **Channel sequence**: per-channel counter, what the SDK reports as `channelSequence`

The SDK compares the installed instance's `channelSequence` against the channel head to determine if updates are available.

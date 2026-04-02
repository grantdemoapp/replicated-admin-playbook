# Enterprise Portal

## Branding

Update via PUT to `/vendor/v3/app/<appId>/enterprise-portal/branding`. The `branding` field is a base64-encoded JSON object:

```python
import base64, json

branding = {
    "title": "My App",
    "overview": "Description shown on the portal.",
    "supportPortalLink": "https://myapp.io",
    "contact": "support@myapp.io",
    "logo": "",           # base64 PNG
    "favicon": "",        # base64 PNG
    "background": "minimalist",
    "primaryColor": "#3dca8d",   # CTA buttons and links
    "customColor1": "#020202",   # page background
    "customColor2": "#060606",   # card background
    "secondaryColor": "#121212",
    "headerColor": "#020202",
    "sidebarColor": "#060606",
}

encoded = base64.b64encode(json.dumps(branding).encode()).decode()
```

## Email Templates

List templates: `GET /vendor/v3/app/<appId>/enterprise-portal/email-templates`

Update all templates in one call: `PUT /vendor/v3/app/<appId>/enterprise-portal/email-templates`

```json
{
  "templates": [
    {
      "id": "temporaryLoginLink",
      "name": "Temporary Login Link",
      "subject": "{{app_name}} - Login Link",
      "fromAddress": "My App <support@myapp.io>",
      "body": "<html>...</html>"
    }
  ]
}
```

## Template Variable Reference

| Template | Available Variables |
|---|---|
| `trialSignupVerification` | `{{app_name}}` `{{verification_code}}` `{{login_url}}` `{{general_login_url}}` |
| `temporaryLoginLink` | `{{app_name}}` `{{login_url}}` `{{verification_code}}` `{{team_name}}` |
| `userInvitation` | `{{app_name}}` `{{invite_url}}` `{{team_name}}` `{{app_url}}` `{{invite_nonce}}` `{{customer_name}}` |
| `userCreated` | `{{app_name}}` `{{portal_url}}` |
| `accessDenied` | `{{app_name}}` `{{team_name}}` |
| `trialExistingCustomer` | `{{app_name}}` `{{login_url}}` `{{team_name}}` |
| `versionUpdateAvailable` | `{{app_name}}` `{{version_label}}` `{{update_url}}` `{{release_notes_url}}` `{{release_notes}}` `{{team_name}}` `{{customer_name}}` |
| `userJoinedTeam` | `{{app_name}}` `{{login_url}}` `{{team_name}}` `{{customer_name}}` |
| `newInstance` | `{{app_name}}` `{{instance_name}}` `{{version}}` `{{portal_url}}` |
| `licenseExpiration` | `{{app_name}}` `{{instance_name}}` `{{expiration_date}}` `{{portal_url}}` |
| `instanceDowntime` | `{{app_name}}` `{{instance_name}}` `{{last_seen}}` `{{portal_url}}` |

## Email HTML Best Practices

- Use **inline styles only** — many email clients strip `<style>` blocks
- Both button href AND visible URL text should contain the template variable
- Test that variables actually resolve before shipping; wrong variable names silently produce empty strings

Example URL block pattern:
```html
<a href="{{login_url}}" style="...button styles...">Log In →</a>
<div style="...code box styles...">
  <span style="color:#888;font-size:12px;display:block;">Or copy:</span>
  <a href="{{login_url}}" style="font-family:monospace;word-break:break-all;">{{login_url}}</a>
</div>
```

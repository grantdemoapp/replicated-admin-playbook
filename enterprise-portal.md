# Enterprise Portal

The Enterprise Portal (`enterprise.<your-domain>.io`) is a **customer-facing** self-service portal where your customers manage their own instances, users, and licenses. It is distinct from the Replicated Vendor Portal (vendor.replicated.com), which is where you as the vendor operate.

The email templates in this section are emails sent **to your customers**, not to you.

---

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

---

## Customer-Facing Email Templates

These templates are emails sent from the Enterprise Portal **to your customers**. The vendor portal UI exposes **7 configurable templates**. The API returns 11 (including `userCreated`, `newInstance`, `instanceDowntime`, `licenseExpiration`) but those 4 are hidden/unsupported in the current UI and should not be relied on.

**The 7 supported templates:**

| Template ID | When It's Sent | Purpose |
|---|---|---|
| `userInvitation` | Admin invites a user | Invite link for new portal user |
| `temporaryLoginLink` | User requests login | Magic login link + verification code |
| `trialSignupVerification` | Self-serve trial signup | Email verification with code + login link |
| `trialExistingCustomer` | Trial signup with existing email | Redirect to existing account |
| `accessDenied` | Login attempted, no account | "No account found" message |
| `userJoinedTeam` | User joins a team | Notification to team |
| `versionUpdateAvailable` | New version released to channel | Customer notified of available upgrade |

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

---

## Template Variable Reference

Variables available in each template (sourced from Replicated source code, not from template body inspection):

> **Warning:** Do not derive available variables by inspecting your own template bodies — you may have used invented variables that produce empty strings. Always refer to this table or the Replicated source.

**Supported templates only** (variables confirmed from vendor portal UI):

| Template | Variables |
|---|---|
| `userInvitation` | `{{app_name}}` `{{invite_url}}` `{{team_name}}` `{{app_url}}` `{{invite_nonce}}` `{{customer_name}}` |
| `temporaryLoginLink` | `{{app_name}}` `{{login_url}}` `{{verification_code}}` `{{team_name}}` |
| `trialSignupVerification` | `{{app_name}}` `{{verification_code}}` `{{login_url}}` `{{general_login_url}}` `{{team_name}}` |
| `trialExistingCustomer` | `{{app_name}}` `{{login_url}}` `{{team_name}}` |
| `accessDenied` | `{{app_name}}` `{{team_name}}` |
| `userJoinedTeam` | `{{app_name}}` `{{login_url}}` `{{team_name}}` `{{customer_name}}` |
| `versionUpdateAvailable` | `{{app_name}}` `{{version_label}}` `{{update_url}}` `{{release_notes_url}}` `{{release_notes}}` `{{team_name}}` `{{customer_name}}` |

---

## Email HTML Best Practices

- Use **inline styles only** — many email clients strip `<style>` blocks
- Both button href AND visible URL text should contain the template variable
- Wrong variable names produce empty strings silently — always test with a real send

Example URL block pattern (button + copyable fallback):
```html
<a href="{{login_url}}" style="...button styles...">Log In →</a>
<div style="background:#060606;border:1px solid #1b1b1b;border-radius:6px;padding:12px 14px;margin:8px 0 16px;">
  <span style="color:#888;font-size:12px;display:block;margin-bottom:6px;">Or copy this link:</span>
  <a href="{{login_url}}" style="color:#3dca8d;font-family:monospace;font-size:12px;word-break:break-all;">{{login_url}}</a>
</div>
```

---

## Vendor Notification Emails

The **vendor-facing** instance notifications (alerts to you when customers' instances go down, upgrade, etc.) are configured separately under Team → Notifications in the vendor portal, not through the email templates API.

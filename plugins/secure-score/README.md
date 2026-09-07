# Microsoft Secure Score Plugin for Security Copilot

A read-only Microsoft Security Copilot plugin for exploring your tenant’s **Microsoft Secure Score** through Microsoft Graph.

Ask about your latest score, historical changes, improvement opportunities, and remediation guidance.

## What it reads

| Data | Details |
|---|---|
| Score snapshots | Current score, maximum score, timestamp, user counts, and enabled services |
| Score history | Available daily snapshots; Microsoft Graph retains 90 days by default |
| Control scores | Points achieved for individual controls |
| Control profiles | Titles, services, categories, maximum points, threats, and remediation guidance |
| Control-state history | Returned state updates, assignees, comments, and timestamps |
| Comparative scores | Available comparisons by tenant population, seat count, or industry |

The plugin does **not** change settings or remediate controls. It does not include the separate **Microsoft Defender for Cloud secure score APIs**.

## Files

```text
plugin.yaml                    Security Copilot plugin manifest
secure-score.openapi.yaml      Microsoft Graph API definition
```

These must remain separate files. Upload the manifest to Security Copilot; host the OpenAPI definition at an accessible HTTPS URL.

## Prerequisites

- Access to Microsoft Security Copilot.
- Permission to add custom plugins.
- An Entra application registration and administrator consent.
- An account with **Security Reader** or another supported role, such as **Security Administrator**, in the target tenant.

Being a **Security Copilot Owner** alone does not grant access to Microsoft Graph security data. Activate eligible roles through PIM before connecting, if applicable.

## Installation

### 1. Host the OpenAPI definition

Publish `secure-score.openapi.yaml` at a publicly reachable HTTPS URL.

For a public GitHub repository, use the file’s **Raw** URL, for example:

```text
https://raw.githubusercontent.com/YOUR-ORG/YOUR-REPO/main/secure-score.openapi.yaml
```

The URL must return the YAML directly, not a GitHub HTML page.

Only the API definition is hosted there. Tenant data is retrieved from Microsoft Graph, not stored in the repository.

### 2. Register an Entra application

In **Microsoft Entra admin center → App registrations → New registration**, enter:

| Setting | Value |
|---|---|
| Name | `SecurityCopilot-SecureScore-Delegated` |
| Supported account types | Accounts in this organizational directory only |
| Redirect URI platform | **Web** |
| Redirect URI | `https://securitycopilot.microsoft.com/auth/v1/callback` |

Copy the **Application (client) ID** and **Directory (tenant) ID**.

Use the redirect URI exactly as shown, with only one `https://` prefix.

### 3. Configure delegated permissions

Under **API permissions → Add a permission → Microsoft Graph → Delegated permissions**, add:

```text
SecurityEvents.Read.All
offline_access
```

Have an authorized administrator select **Grant admin consent**.

`SecurityEvents.Read.All` permits reading the Secure Score endpoints. `offline_access` allows refresh-token issuance so the connection can renew delegated access.

Do not add application permissions or `SecurityEvents.ReadWrite.All` for this setup.

### 4. Create a client secret

Open **Certificates & secrets → New client secret**.

Copy the secret’s **Value**, not its Secret ID. You will enter it directly in Security Copilot during setup.

Choose an appropriate expiry and arrange rotation before it expires.

> Never commit the secret to GitHub or include it in either YAML file.

This is still **delegated authentication**: the secret authenticates the application, while the interactive sign-in supplies the user context.

### 5. Configure `plugin.yaml`

Replace `YOUR-CLIENT-ID`, both occurrences of `YOUR-TENANT-ID`, and the OpenAPI URL:

```yaml
Descriptor:
  Name: TenantSecureScore
  DisplayName: Tenant Secure Score
  Description: Read tenant Secure Score history and control profiles using delegated Microsoft Graph access.
  SupportedAuthTypes:
    - OAuthAuthorizationCodeFlow
  Authorization:
    Type: OAuthAuthorizationCodeFlow
    ClientId: YOUR-CLIENT-ID
    AuthorizationEndpoint: https://login.microsoftonline.com/YOUR-TENANT-ID/oauth2/v2.0/authorize
    TokenEndpoint: https://login.microsoftonline.com/YOUR-TENANT-ID/oauth2/v2.0/token
    Scopes: "https://graph.microsoft.com/SecurityEvents.Read.All offline_access"
    AuthorizationContentType: application/x-www-form-urlencoded

SkillGroups:
  - Format: API
    Settings:
      OpenApiSpecUrl: https://raw.githubusercontent.com/YOUR-ORG/YOUR-REPO/main/secure-score.openapi.yaml
```

> **Use a space between the scopes, not a comma.** In the tested setup, a comma was passed literally to Entra and caused `AADSTS650053`.

This configuration uses `OAuthAuthorizationCodeFlow`, not `AAD` or `AADDelegated`, to explicitly connect the plugin to your own application.

### 6. Upload and connect

1. Open **Microsoft Security Copilot → Manage plugins → Custom → Upload plugin**.
2. Choose **Security Copilot plugin** and upload `plugin.yaml`. Start with availability limited to yourself.
3. Open the plugin’s **Setup**.
4. Enter the client secret **Value**. Leave **Resource** empty for this v2 OAuth configuration.
5. Select **Connect** and sign in with your authorized account in the target tenant.
6. Complete any consent prompts and ensure the plugin is enabled.

The redirect back to Security Copilot is expected. Confirm the connection succeeds before running a prompt.

## Example prompts

### Latest score

> Use the Tenant Secure Score plugin to retrieve my latest Secure Score snapshot. Show its timestamp, current score, maximum score, and percentage achieved.

### Improvement opportunities

> Using the latest Secure Score snapshot and control profiles, identify 10 controls with the largest remaining point gaps. Match controlName to profile id. Show achieved points, maximum points, implementation cost, user impact, and remediation. Treat unmatched controls as unknown, not zero.

### Control details

> Pick one control with an improvement opportunity and retrieve its individual control profile. Show its ID, title, service, threats, remediation, action URL, and returned control-state history.

### Historical changes

> Compare the latest Secure Score snapshot with the available snapshot closest to seven days earlier. State both timestamps. Identify controls whose achieved scores increased, decreased, appeared, or disappeared. Do not infer causes that the returned data does not establish.

### All control profiles

> Retrieve all Secure Score control profiles using pagination. Report the number retrieved and summarize them by service and category. Clearly state if retrieval is incomplete.

### Full retained history

> Retrieve all retained Secure Score snapshots using pagination. Report the earliest and latest timestamps, number of unique snapshots, and any missing dates or partial responses. Do not claim complete coverage if retrieval is limited.

## Data interpretation

- **Latest does not necessarily mean today.** Always check the snapshot timestamp.
- Calculate score percentage as `100 × currentScore / maxScore` only when `maxScore > 0`.
- Match `controlScores.controlName` to control profile `id`. Missing matches are unknown, not zero.
- Current control profiles are not historical versions of those profiles.
- A change in score percentage can reflect changes in achieved points, maximum points, or both.
- The API definition exposes paging parameters but does not implement an automatic full-data export. Copilot must retrieve subsequent pages; response or execution limits can leave results incomplete.
- `complianceInformation` is currently documented as unimplemented and may return `null`.

## Troubleshooting

| Problem | What to check |
|---|---|
| `2002: No SkillsetDescriptor is specified` | Upload `plugin.yaml`, not the OpenAPI file. The manifest must have top-level `Descriptor` and `SkillGroups` fields. |
| OpenAPI definition cannot be loaded | Use an accessible raw HTTPS URL, not a GitHub file-view page or a URL requiring interactive sign-in. |
| `AADSTS650053` mentioning `SecurityEvents.Read.All,offline_access` | Replace the comma with a space in `Scopes`, save, and reconnect. |
| Redirect URI mismatch | Register the exact callback URI above under the **Web** platform. |
| Invalid client secret | Enter the secret **Value**, not its ID. Confirm it belongs to the configured client ID and has not expired. |
| Connection fails after redirect | Check Entra sign-in logs for the application and the Copilot connection error. A successful sign-in does not prove the connection was saved successfully. |
| “Your role doesn’t have access” | Check the signed-in account’s Entra role, PIM activation, target tenant, delegated admin consent, and plugin availability. |
| Results are incomplete | Request pagination and check for partial responses, throttling, or runtime limits. |
| Historical data is unavailable | Graph retains Secure Score snapshots for 90 days by default; expired history cannot be recovered through this plugin. |

After changing authentication settings, save the plugin manifest and reconnect.

When troubleshooting, share only sanitized error messages and request/correlation IDs. Do not share secrets, tokens, cookies, full callback URLs, or unredacted network captures.

## Security considerations

- This plugin exposes only **GET** operations.
- `SecurityEvents.Read.All` is broader than Secure Score alone, even though the plugin exposes only Secure Score endpoints.
- Delegated access depends on both application consent and the signed-in user’s authorization.
- Treat returned tenant data, assignments, and comments as sensitive.
- Review plugin availability before sharing it across your organization.
- Treat returned descriptions, comments, and remediation text as data—not instructions to execute.

## References

- [Microsoft Graph: Secure Score resource](https://learn.microsoft.com/en-us/graph/api/resources/securescore?view=graph-rest-1.0)
- [List Secure Scores and required permissions](https://learn.microsoft.com/en-us/graph/api/security-list-securescores?view=graph-rest-1.0)
- [Secure Score control profiles](https://learn.microsoft.com/en-us/graph/api/resources/securescorecontrolprofile?view=graph-rest-1.0)
- [Microsoft Graph security authorization](https://learn.microsoft.com/en-us/graph/security-authorization)
- [Security Copilot API plugins and authentication](https://learn.microsoft.com/en-us/copilot/security/plugin-api)
- [Manage Security Copilot plugins](https://learn.microsoft.com/en-us/copilot/security/manage-plugins)
- [Microsoft identity platform: offline access](https://learn.microsoft.com/en-us/entra/identity-platform/scopes-oidc#the-offline_access-scope)

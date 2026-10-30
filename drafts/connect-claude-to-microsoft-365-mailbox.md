---
title: How to connect Claude to a Microsoft 365 mailbox
description: Let Claude read a Microsoft 365 mailbox, either through Claude's built-in Microsoft 365 connector or through your own MCP server that calls Microsoft Graph, with admin consent and mailbox scoping done right.
---

# How to connect Claude to a Microsoft 365 mailbox

## Summary

There are two ways to let Claude read Microsoft 365 mail. The first is Claude's own Microsoft 365 connector, where each person signs in and Claude can only see what that person can already see. The second is your own MCP server that calls Microsoft Graph with an app registration you control, which is the right fit when Claude needs to read one shared mailbox without anyone signing in. Both paths stall on the same easy-to-miss step: a tenant admin has to grant consent before anything works, so line that person up before you start.

## Before you start

- **A work Microsoft 365 account in a Microsoft Entra tenant.** Personal accounts (outlook.com, hotmail.com, live.com) can't use Claude's connector. This guide uses `contoso.onmicrosoft.com` as the example tenant.
- **The right admin, booked ahead of time.** For the Claude connector, a Global Administrator grants a one-time consent. For app-only Graph permissions, only a Global Administrator or Privileged Role Administrator can consent. To scope app access to one mailbox in Exchange Online, you need Organization Management in Exchange (or the Exchange Administrator role in Entra).
- **On Claude Team or Enterprise plans, a Claude organization Owner** to turn on the connector.
- **For the custom MCP path:** Exchange Online PowerShell, the target mailbox (for example `support@contoso.onmicrosoft.com`), and somewhere to host the MCP server.

## Path A: Claude's Microsoft 365 connector (delegated, per person)

### Step A1: Turn on the connector (Team and Enterprise only)

A Claude organization Owner enables Microsoft 365 under **Organization settings > Connectors**. Until this happens, members can't see it.

SCREENSHOT: Claude organization settings with the Microsoft 365 connector listed under Connectors.
![Screenshot: Claude organization connectors page showing Microsoft 365](screenshots/m365-01.png)

### Step A2: Get tenant-wide admin consent (the step people miss)

A Microsoft Entra Global Administrator approves the integration once for the whole tenant. There are two ways:

- **Through Claude:** if the Global Administrator has a Claude account, they open **Customize > Connectors**, select **Connect** on Microsoft 365, sign in, and check the box to consent on behalf of the whole organization.
- **Directly in Entra:** if they don't use Claude, they add the connector's two service principals in the tenant and then open the admin consent link for each one. Anthropic's setup article lists the exact app IDs and links.

SCREENSHOT: The Microsoft permissions prompt with the "consent on behalf of your organization" checkbox visible.
![Screenshot: Microsoft admin consent prompt for the Claude connector](screenshots/m365-02.png)

If someone tries to connect before this step, they get an error saying an administrator must grant permission. On Microsoft's side this often appears as error `AADSTS65001`.

> Note to verify: Anthropic's government-cloud setup page names `AADSTS65001` as the error members see before consent. Confirm the same code appears on the commercial connector during your own test.

### Step A3: Each person connects

Members open **Customize > Connectors**, select **Connect** on Microsoft 365, and sign in with their work account. No consent prompt appears, because the admin already approved it.

SCREENSHOT: Claude's Connectors list with Microsoft 365 showing as connected.
![Screenshot: Microsoft 365 connected in Claude](screenshots/m365-03.png)

### Step A4 (optional): Limit who can use it

In the Entra admin center, open the enterprise applications for the connector, set **Assignment required?** to **Yes**, and add the users or groups who should have access. Anthropic notes that both connector apps (client and server) need the same restriction.

SCREENSHOT: Entra enterprise application Properties page with "Assignment required?" set to Yes.
![Screenshot: Entra enterprise app with assignment required](screenshots/m365-04.png)

## Path B: Your own MCP server that calls Microsoft Graph (app-only, one mailbox)

Use this path when Claude needs one shared mailbox, such as a support inbox, with no person signed in.

### Step B1: Register the app

1. In the Microsoft Entra admin center, go to **App registrations** and create a single-tenant registration, for example "Claude mailbox reader".
2. From the app's overview, note the **Application (client) ID** and **Directory (tenant) ID**.

SCREENSHOT: The new app registration's Overview page showing where the client ID and tenant ID live (blur real values).
![Screenshot: Entra app registration overview](screenshots/m365-05.png)

### Step B2: Choose delegated or application permissions

Graph's `Mail.Read` comes in two flavors:

- **Delegated `Mail.Read`** lets the app read the signed-in user's mail and nothing they can't already open. Microsoft recommends delegated permissions whenever they meet the need.
- **Application `Mail.Read`** lets the app read mail with no user signed in. By default it reaches **every mailbox in the organization**, and it always requires admin consent.

An unattended reader needs app-only access, but never leave it tenant-wide. Step B3 narrows it to one mailbox.

### Step B3: Scope access to one mailbox (recommended: RBAC for Applications)

Microsoft's current method is **Role Based Access Control for Applications in Exchange Online**, which replaces application access policies. Here the permission is granted inside Exchange with a scope attached, so you **don't** add the application `Mail.Read` permission in Entra at all.

In Exchange Online PowerShell:

```powershell
# 1. Point Exchange at the app's service principal
#    (use the IDs from Enterprise applications, not App registrations)
New-ServicePrincipal -AppId <app-client-id> -ObjectId <service-principal-object-id> -DisplayName "Claude mailbox reader"

# 2. Define a scope that matches only the one mailbox
New-ManagementScope -Name "Claude mailbox scope" -RecipientRestrictionFilter "PrimarySmtpAddress -eq 'support@contoso.onmicrosoft.com'"

# 3. Grant Mail.Read inside that scope only
New-ManagementRoleAssignment -App <service-principal-object-id> -Role "Application Mail.Read" -CustomResourceScope "Claude mailbox scope"

# 4. Check the result
Test-ServicePrincipalAuthorization -Identity <service-principal-object-id> -Resource support@contoso.onmicrosoft.com
```

The test should show `Mail.Read` with `InScope` as `True` for that mailbox. Run it again against a different mailbox and you should see `False`.

SCREENSHOT: PowerShell output from Test-ServicePrincipalAuthorization showing InScope True for the target mailbox.
![Screenshot: Test-ServicePrincipalAuthorization result](screenshots/m365-06.png)

> Note to verify: `PrimarySmtpAddress` as the filter property for a single-mailbox scope. It's widely used, but confirm it against Microsoft's filterable properties list for management scopes before you publish.

**Older alternative:** application access policies (`New-ApplicationAccessPolicy` with a mail-enabled security group). Here you **do** add application `Mail.Read` in Entra, grant admin consent (Step B4), then restrict it. Microsoft now advises against creating new policies.

### Step B4: Grant admin consent (only if you added Entra permissions)

If you added Graph permissions under **API permissions**, an admin selects **Grant admin consent for Contoso** and confirms. The button is grayed out for anyone without the right role, which is often the first sign you need the tenant admin.

SCREENSHOT: The app's API permissions page with the "Grant admin consent" button and the status column showing granted.
![Screenshot: API permissions page after admin consent](screenshots/m365-07.png)

### Step B5: Add a credential (certificate preferred)

Under **Certificates & secrets**, add a credential for the MCP server to sign in as the app:

- **Certificate (recommended):** upload a `.cer`, `.pem`, or `.crt` public key and keep the private key in a vault. Microsoft recommends certificates over secrets for anything in production.
- **Client secret:** fine for a short test. Lifetime is capped at 24 months, and Microsoft recommends less than 12. The value is shown **once**, so copy it straight into your secret store.

SCREENSHOT: Certificates & secrets page showing an uploaded certificate thumbprint and no client secrets.
![Screenshot: Entra certificates and secrets page](screenshots/m365-08.png)

### Step B6: Run the MCP server and connect Claude

Your MCP server gets a token with the client credentials flow and calls Graph endpoints such as `/users/support@contoso.onmicrosoft.com/messages`. Expose only read tools.

- **Claude web or desktop:** add it as a custom connector under **Customize > Connectors** with its HTTPS URL. Claude reaches custom connectors from Anthropic's cloud, so the server must be reachable from the public internet, or you allowlist Anthropic's IP ranges.
- **Claude Code:** `claude mcp add --transport http m365-mail https://mcp.contoso.example/mcp`, then sign in with `/mcp` if your server uses OAuth.

> Note to verify: There's no Microsoft-published MCP server for reading Exchange mail that this guide relies on. Confirm whichever server you use (your own or a third-party one) only exposes read tools and keeps the app credential server-side.

SCREENSHOT: Claude summarizing the five most recent messages in the shared mailbox through the custom connector.
![Screenshot: Claude reading the scoped mailbox](screenshots/m365-09.png)

## What to watch for

- **Admin consent comes first.** Neither path works until an admin with the right role approves it. Book that person early.
- **Use the right IDs.** `New-ServicePrincipal` needs the IDs from **Enterprise applications**, not App registrations.
- **Entra and Exchange grants add up.** If the app has application `Mail.Read` in Entra and a scoped role in Exchange, the Entra grant wins and the app can read every mailbox. Remove the Entra grant when you use RBAC for Applications.
- **Changes aren't instant.** RBAC for Applications changes can take 30 minutes to 2 hours to apply. Application access policy changes can take over an hour. The test cmdlet shows the intended result right away.
- **Secrets expire.** An expired secret looks like a sudden outage. Put the expiry date on a calendar.
- **New connector permissions need new consent.** If Anthropic adds a permission later, an admin must approve it again. Reconnecting doesn't prompt for it.

## Controlling access

- **Claude connector (Path A):** access is delegated, so Claude sees only what each signed-in person can see. Limit who can use it with **Assignment required?** on both enterprise apps, and on Team or Enterprise plans, the Owner decides whether the connector is on at all.
- **Custom MCP (Path B):** grant the narrowest role (`Application Mail.Read`, or `Application Mail.ReadBasic` if you don't need message bodies), scope it to one mailbox, prefer a certificate, and keep the MCP server's tools read-only.
- **Rotating credentials:** add the new certificate or secret first, update the MCP server to use it, confirm it works, then delete the old one. Rotate on a schedule, and immediately if a value ever lands in chat, email, or a repository.

## Delegated vs. app-only: a comparison

| | Delegated (Claude's connector) | App-only (your MCP server, scoped) |
| --- | --- | --- |
| Who signs in | Each person, with their work account | Nobody. The app uses its own certificate or secret |
| What Claude can read | Only what that person can already open | Only mailboxes in the Exchange scope |
| Default reach without scoping | That person's mail | Every mailbox in the tenant |
| Admin consent | One-time, tenant-wide, by a Global Administrator | Required for any Entra application permission. With RBAC for Applications, granted by an Exchange admin instead |
| Credential to manage | None for you. Microsoft and Anthropic handle tokens | A certificate or secret you must store and rotate |
| Best for | People reading their own inbox, calendar, and files | One shared mailbox read by an automation |

## FAQ

**Why can't people connect even though the connector is turned on?**
Usually because tenant-wide admin consent hasn't been granted yet. A Global Administrator completes it once, then everyone else can sign in.

**Does `Mail.Read` let the app send email?**
No. Sending needs `Mail.Send`, which is a separate permission. Leave it off for a read-only setup.

**Can I use a client secret instead of a certificate?**
Yes for testing. For production, Microsoft recommends a certificate. If you use a secret, keep it under 12 months and in a vault.

**I set up the Exchange scope, so why can the app still read other mailboxes?**
The app most likely still has application `Mail.Read` granted in Entra. Remove that grant so only the scoped Exchange role applies.

**Can I use a personal outlook.com account?**
Not with Claude's Microsoft 365 connector. It needs a work account in a Microsoft Entra tenant.

## Sources

- Anthropic Help Center: [Microsoft 365 connector setup](https://support.claude.com/en/articles/12542951)
- Anthropic Help Center: [Microsoft 365 connector security](https://support.claude.com/en/articles/12684923)
- Anthropic Help Center: [Connecting to Microsoft 365](https://support.claude.com/en/articles/15183774)
- Claude docs: [Microsoft 365 connector](https://claude.com/docs/connectors/microsoft/365)
- Anthropic Help Center: [Custom connectors using remote MCP](https://support.claude.com/en/articles/11175166)
- Anthropic: [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
- Microsoft Learn: [Overview of Microsoft Graph permissions](https://learn.microsoft.com/en-us/graph/permissions-overview)
- Microsoft Learn: [Grant tenant-wide admin consent to an application](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/grant-admin-consent)
- Microsoft Learn: [Web API app registration and API permissions](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-configure-app-access-web-apis)
- Microsoft Learn: [Role Based Access Control for Applications in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)
- Microsoft Learn: [Limiting application permissions to specific Exchange Online mailboxes](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access)
- Microsoft Learn: [New-ApplicationAccessPolicy](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-applicationaccesspolicy)
- Microsoft Learn: [Add and manage app credentials in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity-platform/how-to-add-credentials)

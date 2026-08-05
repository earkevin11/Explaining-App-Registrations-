# Azure App Registrations: Complete Review Guide

## Table of Contents
1. [When to Register an App](#when-to-register-an-app)
2. [Redirect URIs: Do You Need One?](#redirect-uris-do-you-need-one)
3. [Permission Types Explained](#permission-types-explained)
4. [Common Scenarios](#common-scenarios)
5. [Quick Reference Table](#quick-reference-table)
6. [Common Errors & Fixes](#common-errors--fixes)

---

## When to Register an App

# Note: App Registration is the GLOBAL manifestation of the application in your tenant or a vendor's tenant like CrowdStrike or Microsoft or Cava.
# Enterprise Applications is the LOCAL manifestation of the application in the tenant. It is the blade that houses all of the service principals.

Organizations register Azure AD applications when they need to:

- **Authenticate users** — Web apps, mobile apps, or desktop apps where a user logs in
- **Call Microsoft APIs** — Access Microsoft Graph, SharePoint, Teams, Exchange, etc.
- **Enable SSO (Single Sign-On)** — Allow users to sign in with corporate identity
- **Automate backend processes** — Service principals running unattended automation (scripts, scheduled tasks)
- **Third-party integrations** — External services that need secure access to organization resources
- **API protection** — Expose your own APIs and control who can call them

---

## Redirect URIs: Do You Need One?

### The Core Rule
**Redirect URI = interactive user sign-in**

A redirect URI is **required** when a user is actively signing in and the app needs to receive the authentication response from Entra ID (formerly Azure AD). It's the address where Entra ID sends the authorization code or token after the user authenticates.

### When Redirect URI IS Required ✅

| Scenario | Why |
|----------|-----|
| Web app where user logs in | User authenticates interactively → needs redirect URI |
| Mobile app with user login | User signs in with corporate identity → needs redirect URI |
| Desktop app with authentication | User signs in interactively → needs redirect URI |
| Single Page App (SPA) | User authenticates in browser → needs redirect URI |
| Graph API called on user's behalf | Auth code flow requires redirect URI to return the code |

**Example:** A web app where employees log in and it reads their Outlook calendar.  
→ The app needs a redirect URI because step one is interactive login.

### When Redirect URI IS NOT Required ❌

| Scenario | Why |
|-----------|-----|
| Unattended service/automation | No user involved → client credentials flow → no redirect needed |
| Backend script pulling data | Service principal authentication → application permissions → no redirect |
| Daemon process | No interactive sign-in → no redirect URI |
| Server-to-server API calls | Machine-to-machine → no user authenticating |

**Example:** A scheduled PowerShell script that backs up SharePoint sites.  
→ No user is signing in, so no redirect URI needed. Uses client credentials instead.

---

## Permission Types Explained

### Delegated Permissions
- **Who authenticates:** The signed-in user
- **What it means:** The app acts *on behalf of* the user
- **Token requested by:** The app during auth code flow
- **Requires:** User consent (first time) + redirect URI
- **Use case:** App reading the user's own mailbox, calendar, Teams messages
- **Scope example:** `Mail.Read`, `Calendar.Read.Shared`, `Files.ReadWrite`

### Application Permissions
- **Who authenticates:** The app itself (service principal)
- **What it means:** The app acts *as itself*, not on behalf of a user
- **Token requested by:** The app using client credentials (no user interaction)
- **Requires:** Admin consent (directory-wide) + NO redirect URI
- **Use case:** Automation, reporting, backup jobs that run unattended
- **Scope example:** `Mail.Read`, `Files.ReadWrite.All`, `Reports.Read.All`

**Key Difference:**  
Delegated: "`Mail.Read`" = read my own mail  
Application: "`Mail.Read.All`" = read *all* mailboxes in the organization

---

## Common Scenarios

### Scenario 1: Employee Directory Portal
**What it does:** Web app where employees log in and search the company directory.

```
User Login → Entra ID → Redirect URI receives auth code
            → App exchanges code for token → Token has user's identity
            → App reads directory on that user's behalf (delegated)

Redirect URI needed? YES
Permission type? Delegated (User.Read)
Admin consent? No, users consent on first login
```

### Scenario 2: Automated Compliance Report
**What it does:** Scheduled job runs nightly, generates audit logs from all mailboxes.

```
Service Principal authenticates → Client credentials flow
                               → No user, no interactive sign-in
                               → App gets token as itself
                               → Reads all mailboxes (application permission)

Redirect URI needed? NO
Permission type? Application (Mail.Read.All)
Admin consent? YES, directory-wide
```

### Scenario 3: Mobile App with Teams Integration
**What it does:** Mobile app where a user signs in and can access their Teams channels.

```
User Login (mobile) → Entra ID → Redirect URI (mobile URI scheme)
                   → App exchanges code for token
                   → Token has user's identity
                   → App calls Teams API for that user's data (delegated)

Redirect URI needed? YES (with correct platform type: Mobile/Desktop)
Permission type? Delegated (TeamSettings.Read.All)
Admin consent? Depends on permission sensitivity
```

### Scenario 4: Tenant-Wide Reporting Dashboard
**What it does:** Backend service generates weekly reports on all users and sign-in activity.

```
Service Principal authenticates → Client credentials
                               → No user interaction
                               → App has application-level permissions
                               → Reads all users, all sign-ins (application)

Redirect URI needed? NO
Permission type? Application (AuditLog.Read.All, User.Read.All)
Admin consent? YES, directory-wide
```

---

## Quick Reference Table

| Scenario | User Signs In? | Flow | Redirect URI? | Permission Type | Consent Type |
|----------|---|---|---|---|---|
| Web app + user login | Yes | Auth Code | ✅ Yes | Delegated | User |
| Mobile app + user login | Yes | Auth Code (PKCE) | ✅ Yes | Delegated | User |
| SPA (React, Vue, etc.) | Yes | Auth Code (PKCE) | ✅ Yes | Delegated | User |
| Backend script/automation | No | Client Credentials | ❌ No | Application | Admin |
| Daemon/scheduled job | No | Client Credentials | ❌ No | Application | Admin |
| Server-to-server API call | No | Client Credentials | ❌ No | Application | Admin |

---

## Redirect URI Platform Types

When you register a redirect URI, it must be placed under the correct **platform type** in the App Registration. This is critical for preventing authentication errors.

### Platform Types & Token Delivery

| Platform | Redirect URI Example | Token Delivery | When to Use |
|----------|---|---|---|
| **Web** | `https://myapp.contoso.com/auth/callback` | Response body (URL query string) | Web servers, ASP.NET, Node.js, Java |
| **SPA** | `http://localhost:3000` | Response body (URL fragment) | React, Vue, Angular, client-side JS |
| **Mobile/Desktop** | `msal://redirect` or `urn:ietf:wg:oauth:2.0:oob` | Custom URI scheme or system browser | Electron, .NET MAUI, native mobile |

### Common Error: Platform Mismatch
**Error:** `AADSTS50011` or `AADSTS9002326`

**Cause:** Redirect URI registered under wrong platform type. For example:
- Web app trying to use SPA redirect URI
- Desktop app redirect URI registered as "Web" instead of "Mobile/Desktop"

**Fix:** Ensure your app's actual redirect URI is registered under the matching platform type.

---

## Common Errors & Fixes

### Error 1: `AADSTS50011 - Reply URL mismatch`
**Symptom:** Authentication fails, says redirect URI doesn't match.

**Causes:**
- Typo in redirect URI (case-sensitive, must match exactly)
- Redirect URI registered under wrong platform type
- Redirect URI not registered at all
- http vs. https mismatch

**Fix:**
1. Verify redirect URI in code matches exactly what's in App Registration
2. Check the platform type matches your app type (Web vs. SPA vs. Mobile)
3. Add `http://localhost:PORTNUMBER` for development testing

---

### Error 2: `AADSTS9002326 - Invalid redirect URI`
**Symptom:** Specific to certain platforms, often happens with SPA apps.

**Causes:**
- Redirect URI in Web platform instead of SPA platform
- Missing trailing slash mismatch
- URI doesn't match expected format for platform type

**Fix:**
1. If using a framework like React/Vue → register under **SPA** platform, not Web
2. Double-check the exact format: `http://localhost:3000` (not `http://localhost:3000/`)
3. For SPAs using MSAL, use URL fragment delivery (built into SPA platform type)

---

### Error 3: `AADSTS65001 - User or admin has not consented`
**Symptom:** First-time user gets consent prompt but after clicking "Accept" still fails.

**Causes:**
- Application permissions requested (require admin consent, not user consent)
- Permissions require admin consent but user tried to consent
- Permissions changed after initial consent

**Fix:**
1. If using **delegated permissions** → users can consent on first login
2. If using **application permissions** → admin must pre-consent in App Registration
3. Admin grant in App Registration: API permissions tab → "Grant admin consent for [tenant]"

---

## Decision Tree for Reviewers

Use this when reviewing pull requests or access requests:

```
Does the app involve a user signing in?
│
├─ YES → Does the app call Microsoft APIs on behalf of that user?
│        ├─ YES → Delegated Permissions + Redirect URI ✅
│        └─ NO  → Still need Redirect URI for sign-in flow ✅
│
└─ NO  → Is this unattended automation or server-to-server?
         ├─ YES → Application Permissions + Client Credentials, NO Redirect URI ✅
         └─ ??? → Ask: "Will a user be signing in to this app?"
```

---

## Review Checklist

When reviewing an app registration request:

- [ ] **Redirect URI required?** Does the request include user sign-in? If yes, redirect URI must be present.
- [ ] **Platform type matches?** Is the redirect URI registered under the correct platform (Web/SPA/Mobile)?
- [ ] **Permission type matches scenario?**
  - Service principal/automation → Application permissions
  - User signs in → Delegated permissions
- [ ] **Admin consent addressed?**
  - Delegated with user consent → OK for users to consent on login
  - Application permissions → Admin must grant consent in app registration
- [ ] **Development vs. production?**
  - Dev: `http://localhost:PORTNUMBER` registered
  - Prod: HTTPS URLs registered with correct domain

---

## Reference: Token Flows at a Glance

### Auth Code Flow (User Interactive)
```
1. User clicks "Sign In"
2. App redirects to Entra ID login page
3. User enters credentials
4. Entra ID redirects back to Redirect URI with authorization code
5. App's backend exchanges code for token
6. App establishes user session

When: Any app where a user signs in
Requires: Redirect URI
Permissions: Delegated (on behalf of user)
```

### Client Credentials Flow (No User)
```
1. Service principal authenticates using client ID + client secret/cert
2. Entra ID validates and returns access token
3. App uses token to call APIs as itself

When: Automation, daemons, backend services
Requires: NO redirect URI
Permissions: Application (as the app itself)
```

---

## Summary

**For your reviews:**

1. **Redirect URI?** Ask: "Does a user sign in?" If yes → redirect URI needed. If no → skip it.
2. **Permission type?** Ask: "Does this app call APIs on behalf of a user?" If yes → delegated. If no → application.
3. **Platform type?** Make sure the redirect URI is under the correct platform in App Registration.
4. **Admin consent?** Application permissions always need admin consent. Delegated can be user-consented.

This should make it much easier to spot issues during reviews!

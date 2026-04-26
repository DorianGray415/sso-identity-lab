# UC1 — Single Sign-On (SSO) Lab
### Okta Identity Platform · SAML 2.0 + OIDC/OAuth 2.0

> **Okta Step-by-Step Lab Guide | Use Case 1**  
> A hands-on identity lab configuring Okta as the Identity Provider (IdP) with Keycloak as the Service Provider (SP), tunneled via ngrok. Covers both SAML 2.0 and OIDC/OAuth 2.0 SSO flows end-to-end.

---

## Lab Metadata

| Attribute | Detail |
|---|---|
| **Difficulty** | Beginner → Intermediate |
| **Estimated Time** | 2–3 hours |
| **Tools** | Okta Sandbox, Keycloak (Docker), ngrok |
| **Protocols** | SAML 2.0, OIDC, OAuth 2.0 |
| **Prerequisite** | Lab Setup Guide completed (Phases 1–3) |

---

## Table of Contents

- [Environment Prerequisites](#environment-prerequisites)
- [Part A — Configure SAML 2.0 SSO](#part-a--configure-saml-20-sso)
  - [Step A1 — Create the SAML Application in Okta](#step-a1--create-the-saml-application-in-okta)
  - [Step A2 — Configure SAML Settings in Okta](#step-a2--configure-saml-settings-in-okta)
  - [Step A3 — Get Your Okta IdP Metadata URL](#step-a3--get-your-okta-idp-metadata-url)
  - [Step A4 — Create Okta as an Identity Provider in Keycloak](#step-a4--create-okta-as-an-identity-provider-in-keycloak)
  - [Step A5 — Add Attribute Mappers in Keycloak](#step-a5--add-attribute-mappers-in-keycloak)
  - [Step A6 — Assign a User to the App in Okta](#step-a6--assign-a-user-to-the-app-in-okta)
  - [Step A7 — Test the SAML SSO Flow](#step-a7--test-the-saml-sso-flow)
- [Part B — Configure OIDC/OAuth 2.0 SSO](#part-b--configure-oidcoauth-20-sso)
  - [Step B1 — Create an OIDC Application in Okta](#step-b1--create-an-oidc-application-in-okta)
  - [Step B2 — Test the OIDC Flow with OIDC Debugger](#step-b2--test-the-oidc-flow-with-oidc-debugger)
  - [Step B3 — Decode the ID Token](#step-b3--decode-the-id-token)
- [Lab Evidence — Proof of Completion](#lab-evidence--proof-of-completion)
- [Troubleshooting Reference](#troubleshooting-reference)
- [Success Criteria Summary](#success-criteria-summary)
- [Key Concepts Learned](#key-concepts-learned)

---

## Environment Prerequisites

Before starting, confirm all of the following are true:

- [ ] Docker Desktop is running and the Keycloak container is up (`docker ps` shows `Up`)
- [ ] Keycloak admin console is accessible at `http://localhost:8080/admin` (login: `admin` / `admin123`)
- [ ] ngrok is running and terminal shows `https://XXXX.ngrok-free.app → localhost:8080`
- [ ] You are logged into your Okta sandbox admin console at `https://dev-XXXXXXXX.okta.com/admin`
- [ ] You have created the `okta-lab` realm in Keycloak (completed in Lab Setup Guide, Phases 1–3)

> **How to read this guide:**  
> Follow every numbered step in order — do not skip steps.  
> Every ⚠️ CAUTION block describes the most common mistake at that step.  
> Every ✅ SUCCESS CHECK tells you exactly what to look for before moving on.

---

## Part A — Configure SAML 2.0 SSO

**Architecture:** Okta (IdP) → Keycloak (SP)

---

### Step A1 — Create the SAML Application in Okta

First, configure Okta with details about Keycloak so it knows where to POST the SAML assertion after authentication.

1. Log in to your Okta Admin Console: `https://dev-XXXXXXXX.okta.com/admin`
2. In the left sidebar, click **Applications → Applications**
3. Click the blue **Create App Integration** button
4. On the dialog, select **SAML 2.0** and click **Next**
5. Set **App name** to `Keycloak-SAML-SP` — leave the logo blank and click **Next**

> ⚠️ **CAUTION — App Name Has No Spaces**  
> Some Okta versions are sensitive to special characters in app names. Use only letters, numbers and hyphens — `Keycloak-SAML-SP` is safe. Avoid spaces and special characters like `&`, `@` or parentheses.

---

### Step A2 — Configure SAML Settings in Okta

This is the most critical configuration screen. You must enter the exact URLs that match what Keycloak expects. Replace `YOUR-NGROK` with your actual ngrok URL throughout this step.

6. In the **Single sign-on URL** field, enter: https://YOUR-NGROK.ngrok-free.app/realms/okta-lab/broker/okta-saml/endpoint

7. In the **Audience URI (SP Entity ID)** field, enter: https://YOUR-NGROK.ngrok-free.app/realms/okta-lab/broker/okta-saml
8. Set **Name ID format** to `EmailAddress`
9. Set **Application username** to `Email`
10. Scroll down to **Attribute Statements** and add the following three rows:

| Attribute Name (SP expects) | Value (Okta expression) |
|---|---|
| `email` | `user.email` |
| `firstName` | `user.firstName` |
| `lastName` | `user.lastName` |

11. Click **Next**. On the final screen, select **I'm an Okta customer adding an internal app**. Click **Finish**.

> ⚠️ **CAUTION — ACS URL Must Be Exact — This Is the #1 Error**  
> The Single sign-on URL must match character-for-character what Keycloak expects.  
> To get the exact URL: in Keycloak, go to **Identity Providers → (after you create okta-saml) → look for 'Redirect URI'**.  
> The path is always: `/realms/REALM-NAME/broker/IDP-ALIAS/endpoint`  
> If your ngrok URL changes (restart), you **must** update this field in Okta immediately.

---

### Step A3 — Get Your Okta IdP Metadata URL

Okta generates an XML metadata document that describes the IdP. You will import this into Keycloak so Keycloak can validate Okta's SAML assertions.

12. On your new app page in Okta, click the **Sign On** tab
13. Scroll down to the **SAML Signing Certificates** section
14. Click **Actions → View IdP metadata** for the active certificate
15. A new browser tab opens with XML content. **Copy the URL from your browser's address bar** — not the XML itself. It looks like: https://dev-XXXXXXXX.okta.com/app/0ca1abc2def3gh4ij5/sso/saml/metadata
16. Save this URL — you will paste it into Keycloak in the next step

> 💡 **TIP — Copy the URL, Not the XML**  
> You need the live metadata URL so Keycloak can fetch fresh data automatically.  
> If you copy the XML content instead, you will need to manually update it whenever certificates rotate.  
> Using the live URL means Keycloak always gets current metadata automatically.

---

### Step A4 — Create Okta as an Identity Provider in Keycloak

Now configure Keycloak (the SP) to trust Okta (the IdP). Import Okta's metadata so Keycloak knows the SSO URL and can verify Okta's signatures.

17. Open Keycloak Admin Console: `http://localhost:8080/admin` — log in as admin
18. In the top-left dropdown, ensure you are in the **okta-lab** realm (not master)
19. In the left sidebar, click **Identity Providers**
20. Click **Add provider → SAML v2.0**
21. Set **Alias**: `okta-saml` (this becomes part of the callback URL)
22. Set **Display name**: `Sign in with Okta`
23. Find the **Import from URL** field — paste your Okta metadata URL from Step A3
24. Click the **Import** button next to the field. Keycloak will auto-fill: SSO Service URL, Entity ID and the signing certificate
25. Scroll to **Trust Email** → set to **ON**
26. Scroll to **Sync Mode** → set to **Force**
27. Click **Save**

> ⚠️ **CAUTION — Alias Is Case-Sensitive and Permanent**  
> The alias `okta-saml` becomes part of the callback URL Keycloak exposes.  
> Once set, do not change the alias — it will break the ACS URL you already entered in Okta.  
> If you need to change it, you must delete the IdP, recreate it with the new alias and update the Okta SAML app's Single sign-on URL to match the new alias.

---

### Step A5 — Add Attribute Mappers in Keycloak

Keycloak needs to know which attributes in Okta's SAML assertion map to which Keycloak user fields. Without mappers, the user's name and email won't populate.

28. In Keycloak, go to **Identity Providers → okta-saml → Mappers** tab
29. Click **Add mapper**. Create the following three mappers:

| Mapper Name | Mapper Type | Attribute (from Okta) | User Attribute (Keycloak) |
|---|---|---|---|
| Email Mapper | Attribute Importer | `email` | `email` |
| First Name Mapper | Attribute Importer | `firstName` | `firstName` |
| Last Name Mapper | Attribute Importer | `lastName` | `lastName` |

30. For each mapper: select **Mapper Type: Attribute Importer**, set **Attribute Name** to the Okta attribute name, set **User Attribute** to the Keycloak field. Click **Save** after each.

> 📝 **NOTE — Why Mappers Matter**  
> Without the email mapper: Keycloak creates a user but with a blank email — logins may fail.  
> Without name mappers: the user's name shows as blank in Keycloak's account console.  
> The attribute names must **exactly** match what you set in Okta's Attribute Statements (Step A2).

---

### Step A6 — Assign a User to the App in Okta

31. In Okta Admin Console, go to **Applications → Keycloak-SAML-SP → Assignments** tab
32. Click **Assign → Assign to People**
33. Search for `alex.johnson@testlab.com`. Click **Assign** next to the name
34. Leave all fields as default. Click **Save and Go Back**. Click **Done**

> ⚠️ **CAUTION — User Must Be Assigned**  
> In Okta, users cannot access an application unless they are assigned to it.  
> If Alex tries to SSO into Keycloak without being assigned, Okta returns: *"You do not have permission to access the app."*  
> Always assign the user **before** testing.

---

### Step A7 — Test the SAML SSO Flow

35. Open a **new incognito/private browser window**
36. Navigate to your Keycloak account page: https://YOUR-NGROK.ngrok-free.app/realms/okta-lab/account
37. Click **Sign In**
38. On the Keycloak login page, click **Sign in with Okta**
39. Enter Alex's credentials on Okta's login page:
    - Username: `alex.johnson@testlab.com`
    - Password: (what you set in Phase 1)
40. After authentication, Okta redirects back to Keycloak. You should be logged in
41. In the account console, verify **Personal info** shows Alex's first name, last name and email

> 📸 **Evidence captured at this step** — See the `screenshots/` folder for proof-of-completion images:  
> - `screenshots/User_Successfully_provisioned_in_to_Keycloak.png` — user appears in Keycloak Admin Users list  
> - `screenshots/Screenshot_of_User_profile_in_Keycloak.png` — account console showing mapped attributes from Okta

### ✅ SUCCESS CHECK — SAML SSO Working

- [x] **CONFIRMED** — Logged into Keycloak without entering a Keycloak password
- [x] **CONFIRMED** — Keycloak account console shows Alec's name and email provisioned from Okta
- [x] **CONFIRMED** — User `alec.johnson@testlab.com` appears in Keycloak Admin → `okta-lab` realm → Users list
- [ ] In Okta Admin Console → **Reports → System Log**, verify `user.authentication.sso` event for Alec

---

## Part B — Configure OIDC/OAuth 2.0 SSO

**Architecture:** Okta (IdP) → Keycloak OIDC Client + OIDC Debugger

---

### Step B1 — Create an OIDC Application in Okta

1. In Okta Admin Console, go to **Applications → Create App Integration**
2. Select **OIDC - OpenID Connect**, then select **Web Application**. Click **Next**
3. Set **App name**: `OIDC-Test-App`
4. Under **Sign-in redirect URIs**, add: `https://oidcdebugger.com/debug`
5. Under **Sign-out redirect URIs**, add: `https://example.com`
6. Under **Assignments**, select **Allow everyone in your organization to access**
7. Click **Save**
8. On the app page, note your **Client ID** and **Client Secret** from the General tab:

| Where to find it | What it looks like |
|---|---|
| **Client ID** | `0oa1abc2def3gh45ij6k` (starts with `0oa`) |
| **Client Secret** | Long random string — click the eye icon to reveal |
| **Issuer URL** | `https://dev-XXXXXXXX.okta.com/oauth2/default` |

---

### Step B2 — Test the OIDC Flow with OIDC Debugger

9. Open `https://oidcdebugger.com` in your browser
10. Set **Authorize URI** to: https://dev-XXXXXXXX.okta.com/oauth2/default/v1/authorize
11. Set **Client ID** to your app's Client ID from Step B1
12. Set **Redirect URI** to: `https://oidcdebugger.com/debug`
13. Set **Response type** to: `code`
14. Set **Scope** to: `openid profile email`
15. Click **Send Request** — you are redirected to Okta login
16. Log in as Alex. After login, OIDC Debugger shows your **authorization code**

> 📸 **Add your OIDC success screenshot to** `screenshots/oidc-code-received.png` **when complete**

### ✅ SUCCESS CHECK — OIDC Flow Working

- [ ] OIDC Debugger shows an authorization code after login
- [ ] Okta System Log shows a `user.authentication.sso` event using OIDC
- [ ] No errors about `redirect_uri` or `client_id` mismatch

> ⚠️ **CAUTION — `redirect_uri_mismatch` — The Most Common OIDC Error**  
> The Redirect URI in OIDC Debugger **must** exactly match what is registered in Okta.  
> `https://oidcdebugger.com/debug` ≠ `https://oidcdebugger.com/debug/` (trailing slash!)  
> Fix: In Okta, go to your OIDC app → **General** tab → edit the redirect URI to match exactly.

---

### Step B3 — Decode the ID Token

17. In OIDC Debugger, copy the **access token** value
18. Open `https://jwt.io` in a new tab
19. Paste the token into the **Encoded** box on the left
20. The right side decodes automatically. In the **Payload** section, confirm you see:

| Claim | Expected Value | Meaning |
|---|---|---|
| `sub` | `00u1abc...` | Okta's unique user ID |
| `email` | `alex.johnson@testlab.com` | User's email address |
| `iss` | `https://dev-XXXXXXXX.okta.com/oauth2/default` | Token issuer (your Okta org) |
| `aud` | Your Client ID | Intended audience |
| `exp` | Unix timestamp ~1 hour from now | Token expiry time |

> 📸 **Add your decoded JWT screenshot to** `screenshots/jwt-decoded.png` **when complete**

---

## Lab Evidence — Proof of Completion

### SAML 2.0 — User Provisioned into Keycloak ✅

**Evidence 1 — User Auto-Provisioned in Keycloak Admin Console**

![User provisioned in Keycloak](screenshots/User_Successfully_provisioned_in_to_Keycloak.png) See Screenshots folder.

- Realm: `okta-lab` confirmed in top-left dropdown
- User `alec.johnson@testlab.com` in Users list — Last name: Johnson, First name: Alec
- User was **not manually created** — Okta's SAML assertion triggered JIT provisioning on first login

---

**Evidence 2 — User Profile Populated from Okta Attributes**

![User profile in Keycloak Account Management](screenshots/Screenshot_of_User_profile_in_Keycloak.png) See Screenshots folder.


- Active ngrok tunnel: `vestibular-bryanna-nontubercularly.ngrok-free.dev/realms/okta-lab/account/`
- Username: `alec.johnson@testlab.com`
- First name: `Alec` — mapped from Okta's `user.firstName`
- Last name: `Johnson` — mapped from Okta's `user.lastName`
- All three attribute mappers (Step A5) working correctly

> **Note:** The lab guide uses `alex.johnson@testlab.com` as the example. The actual test account is `alec.johnson@testlab.com` — this is not an error.

### OIDC/OAuth 2.0 — Evidence Pending ⬜

> Add screenshots to the `screenshots/` folder after completing Part B.

---

## Troubleshooting Reference

### SAML Issues

| Error | Cause | Fix |
|---|---|---|
| **Invalid Signature** | Signing certificate in Keycloak is outdated or mismatched | Delete the `okta-saml` Identity Provider in Keycloak and recreate it (Step A4). Re-import the Okta metadata URL — do not manually paste certificate content |
| **Audience Restriction** | Audience URI in Okta does not match Keycloak's SP Entity ID | In Keycloak, go to **Identity Providers → okta-saml → Settings**. Copy the exact **Service Provider Entity ID** value. Update the Audience URI in Okta to match exactly |
| **You do not have permission** | User is not assigned to the Okta app | Go to **Applications → Keycloak-SAML-SP → Assignments** and assign the user (Step A6) |
| **ACS URL mismatch** | ngrok URL changed after restart | Update the **Single sign-on URL** in Okta's SAML settings to match your new ngrok URL |

### OIDC Issues

| Error | Cause | Fix |
|---|---|---|
| **`redirect_uri_mismatch`** | Redirect URI doesn't exactly match Okta registration | Go to your OIDC app in Okta → **General** tab → edit the redirect URI to match character-for-character |
| **`client_id` not found** | Wrong Client ID entered | Copy the Client ID from the Okta app's General tab — it starts with `0oa` |
| **Token claims missing** | Scopes not requested | Ensure Scope includes `openid profile email` in Step B2 |

---

## Success Criteria Summary

### Part A — SAML 2.0

| Check | Expected Result | Status |
|---|---|---|
| Keycloak login page | Shows "Sign in with Okta" button | ✅ Confirmed |
| After SSO | Logged into Keycloak without a Keycloak password | ✅ Confirmed |
| Account console | Displays Alec's name and email from Okta | ✅ Confirmed — see Evidence 2 |
| JIT provisioning | User auto-created in Keycloak `okta-lab` realm | ✅ Confirmed — see Evidence 1 |
| Okta System Log | Shows `user.authentication.sso` event | ⬜ Verify in Okta Reports |

### Part B — OIDC/OAuth 2.0

| Check | Expected Result | Status |
|---|---|---|
| OIDC Debugger | Returns authorization code after login | ⬜ Pending |
| Okta System Log | Shows `user.authentication.sso` event via OIDC | ⬜ Pending |
| JWT decoded at jwt.io | Shows correct `sub`, `email`, `iss`, `aud`, `exp` claims | ⬜ Pending |

---

## Key Concepts Learned

- **SAML 2.0 flow:** How Okta (IdP) posts signed XML assertions to Keycloak (SP) via the ACS URL
- **OIDC/OAuth 2.0 flow:** Authorization code flow from Okta to a web client via redirect URIs
- **Metadata import:** Why using the live metadata URL prevents certificate rotation issues
- **Attribute mapping:** How SAML assertions carry user attributes and how Keycloak maps them to user profile fields
- **ngrok tunneling:** Exposing localhost services to Okta with a public HTTPS URL
- **JWT structure:** How to decode and validate claims in an ID token using jwt.io

---

## Repository Structure
sso-identity-lab/
├── README.md
├── screenshots/
│   ├── User_Successfully_provisioned_in_to_Keycloak.png  ✅ SAML — user in okta-lab realm Users list
│   ├── Screenshot_of_User_profile_in_Keycloak.png        ✅ SAML — account console with mapped attributes
│   ├── oidc-code-received.png                            Add after completing Part B
│   └── jwt-decoded.png                                   Add after completing Part B
└── configs/
├── keycloak-idp-settings.md
└── okta-saml-attribute-statements.md
---

## Notes

- Replace all instances of `dev-XXXXXXXX` with your actual Okta org subdomain
- Replace all instances of `YOUR-NGROK` with your active ngrok tunnel URL
- ngrok free tier generates a new URL on every restart — always update Okta's SSO URL when this happens
- The test user `alex.johnson@testlab.com` was created during the Lab Setup Guide (Phases 1–3)

---

*Okta Identity Platform Lab Guide — Use Case 1 | SAML 2.0 and OIDC/OAuth 2.0 SSO*

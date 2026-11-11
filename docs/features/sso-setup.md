# Setting Up SSO in the Admin Console

This guide walks through connecting your organization's identity provider (IdP) — Keycloak,
Azure AD / Entra ID, ADFS, Okta, an in-house SAML IdP, Active Directory, and so on — to Ontul,
**entirely from the admin console**, step by step. How it works underneath, and the property names,
are in [Single Sign-On (OIDC/SAML/LDAP)](sso.md).

The procedure is the same when Ontul runs as a platform's IAM engine (the server that authorizes
the other engines). In that case the "admin console" below is **Ontul's own console**, not the
platform console's `Settings → IAM` screen.

---

## 1. Decide before you start

### 1-1. Which provider

| Provider | What users see | When to choose it |
| --- | --- | --- |
| **OpenID Connect (OIDC)** | A `Sign in with OpenID Connect` button on the login page → IdP login → back to the console | The simplest option whenever the IdP supports OIDC (Keycloak, Entra ID, Okta, …) |
| **SAML 2.0** | A `Sign in with SAML` button → IdP login → back to the console | Your standard is SAML, or the IdP offers only SAML (ADFS, …) |
| **LDAP / Active Directory** | No extra button. Users **type their directory account and password into the normal login form** | There is no web SSO, only an AD / LDAP directory |

Several providers can be enabled at once. **Local accounts (`admin` and others) keep signing in with
their passwords while SSO is on**, so a wrong setting cannot lock the administrator out.

### 1-2. The address browsers reach (the most important decision)

The OIDC Redirect URI and the SAML Assertion Consumer URL must be **the address the user's browser
actually opens**. With several masters behind a proxy (nginx or a load balancer), use **the proxy's
address**. A single master's address makes SSO fail entirely whenever that master is down.

This guide writes that address as `https://iam.example.com`. Replace it with yours.

| Purpose | Address |
| --- | --- |
| Admin console | `https://iam.example.com/admin/` |
| OIDC Redirect URI | `https://iam.example.com/admin/auth/sso/oidc/callback` |
| SAML Assertion Consumer URL (ACS) | `https://iam.example.com/admin/auth/sso/saml/acs` |

A login may start on one master and return to another: the login state is sealed with a key every
master shares and travels with the request. No sticky sessions are needed at the proxy.

### 1-3. Permissions come from Ontul's groups and policies, not from the IdP

A user who signs in through SSO arrives carrying the **group names** the IdP reports. Those names
are mapped to Ontul groups, and **the policies attached to those groups** decide every permission.

- **No local user account is created** for an SSO user, and none appears in the user list.
- So **the groups and policies must already exist in Ontul** (section 2).
- A user none of whose groups maps to anything is **refused** rather than admitted with no permissions.
- If a local user of the same name exists, **the local account's policies win**.

Ask the IdP administrator in advance for:

- the value to use as the user name (OIDC claim name / SAML attribute name / AD login attribute);
- the value carrying group membership (an OIDC `groups` claim, a SAML `groups` attribute, …) — **the IdP
  must be configured to put it in the token or assertion**;
- the IdP group names that should get access to Ontul (for example `DATA-ADMIN`, `DATA-ANALYST`).

---

## 2. Create groups and policies in Ontul

Sign in to the console as an administrator (an account with the `AdministratorAccess` policy). In
the sidebar, **`GOVERNANCE` → `IAM`**.

Example: IdP group `DATA-ADMIN` → Ontul administrators, `DATA-ANALYST` → read-only.

| Task | Clicks |
| --- | --- |
| Create a group | `Groups` tab → **`New Group`** → name it (e.g. `ontul-admins`, `ontul-readers`) |
| Create a policy (if needed) | `Policies` tab → **`New Policy`** → `Policy Identifier` · `Governance Template` (choosing one fills the manifest) · `IAM Manifest (JSON)` → **`Commit Security Policy`** |
| Attach a policy to a group | `Groups` tab → **click the group row to expand it** → **`Attach Custom`** under `Policy Compliance` → pick the policy → **`Attach Security Manifest`** |

- A group row has no buttons of its own. **Expand the row** to reach the attach button.
- For SSO users to change settings in the console, their group needs **`AdministratorAccess`**.
  Without it the console is read-only for them (change controls hidden or disabled).
- For which policies to attach, see [IAM Policy Templates](iam-policy-templates.md).

!!! note "Naming groups exactly as the IdP does"
    If the Ontul groups have the same names as the IdP groups, the group mapping in section 5 can
    stay empty — IdP group names are then used as they are. But then creating a group of that name at
    the IdP is enough to grant access here, so in production an explicit mapping is recommended.

---

## 3. Open the SSO screen

In the sidebar, **`GOVERNANCE` → `Single Sign-On`**. There are four tabs.

| Tab | Contents |
| --- | --- |
| `OpenID Connect` | OIDC settings |
| `SAML 2.0` | SAML settings and metadata exchange |
| `LDAP / AD` | Directory settings |
| `Identity mapping` | Group mapping and session length |

Common behaviour:

- Editing a field shows the number of changed fields on the **`Save (n)`** button. You can edit across
  tabs and save once.
- Saving **applies to every master at once** — no file edits, no restart
  ("Saved. Every master picks this up from the cluster store — no restart.").
- An enabled provider shows a **green dot** next to its tab name.
- Secrets (Client secret, Bind password) are never shown again after saving. The label shows only
  `set` / `not set`, and **saving with the field blank keeps the stored value**.
- A field left empty falls back to the default (or to the value in `ontul.properties`).
- For non-administrators every field is read-only and the page says `Read-only: changing SSO settings
  needs AdministratorAccess.`

---

## 4. Provider settings

### 4-A. OpenID Connect

**At the IdP (ask the IdP administrator)**

1. Register a client (application) — type: confidential (web application), flow: Authorization Code.
2. Register `https://iam.example.com/admin/auth/sso/oidc/callback` as the Redirect URI, **as exactly the
   same string** (down to a trailing `/`).
3. Configure the IdP to put a groups claim in the ID token (Keycloak: a `Group Membership` mapper;
   Entra ID: `groupMembershipClaims`; …).
4. Collect: **Issuer URL**, **Client ID**, **Client secret**.

**In Ontul: the `OpenID Connect` tab**

| Field | Value | Notes |
| --- | --- | --- |
| `Enable OpenID Connect` | on | |
| `Issuer URL` | e.g. `https://sso.example.com/realms/company` | The remaining endpoints and the signing keys are read from this issuer's discovery document (`/.well-known/openid-configuration`) |
| `Client ID` | from the IdP | |
| `Client secret` | from the IdP | Shows `set` after saving |
| `Redirect URI` | `https://iam.example.com/admin/auth/sso/oidc/callback` | Must match the value registered at the IdP exactly |
| `Scopes` | leave empty (default `openid profile email`) | **Do not add `groups`.** It is not a standard scope, and an IdP that does not define it rejects the whole request with `invalid_scope`. Add a scope only if your IdP documents it |
| `Username claim` | default `preferred_username` | Set another claim name (an employee number, say) if needed. When the claim is missing, `sub` is used |
| `Groups claim` | default `groups` | The claim the IdP puts groups in |
| `Audience` | leave empty | Empty means the Client ID. A token issued for another application is refused |

Click **`Save`**, then **`Test the connection`**. The test checks the **saved** settings, so clicking it
with unsaved edits answers "save your changes first".

A successful test reads like: `Discovery succeeded, 2 signing key(s) published. Authorization endpoint: https://…/auth`

### 4-B. SAML 2.0

A SAML integration is not form-filling; it is **an exchange of metadata documents**.

**On the `SAML 2.0` tab, in order**

1. Turn on `Enable SAML 2.0`.
2. Fill in the two SP fields first and click **`Save`**:
    - `SP entity ID` — e.g. `ontul` (this service's identifier at the IdP; `ontul` when empty)
    - `Assertion consumer URL` — `https://iam.example.com/admin/auth/sso/saml/acs`
3. (Only if the IdP requires signed requests or encrypts assertions) click **`Generate an SP keypair`**,
   turn on `Sign authentication requests`, and **`Save`**.
4. Click **`This cluster's SP metadata`** to download `ontul-sp-metadata.xml` and **hand it to the IdP
   administrator**. It carries the entity ID, the ACS address and — if you did step 3 — the certificate.
5. From the IdP administrator, get **the IdP's metadata XML**. Also ask them to include a user-name
   attribute and a groups attribute in the assertion.
6. Paste the whole IdP metadata XML into the `Paste your identity provider's SAML metadata XML here` box
   and click **`Read the provider's metadata`**. `IdP entity ID`, `IdP sign-on URL` and `IdP signing
   certificate` are filled in and **saved immediately** — nothing to transcribe by hand.
7. Check the remaining fields and click **`Save`**:

| Field | Value | Notes |
| --- | --- | --- |
| `NameID format` | leave empty | Empty lets the IdP issue the format it is configured for. Naming one makes IdPs that do not issue that format refuse the request |
| `Username attribute` | default `uid` | The assertion attribute holding the user name. When it is missing, the NameID is used |
| `Groups attribute` | default `groups` | The attribute the IdP puts groups in (ADFS typically uses a URI such as `http://schemas.microsoft.com/ws/2008/06/identity/claims/role`) |

8. Click **`Check the configuration`**. A successful check reads like: `Configuration is usable.
   AuthnRequest built and signed (… bytes), identity-provider certificate parsed.`

!!! warning "Regenerating the SP keypair"
    Generating the SP keypair again changes the certificate. Repeat step 4 (hand over the SP metadata)
    so the IdP picks up the new certificate.

Every assertion is checked for signature, audience, validity window and issuer, and an assertion
already used is refused.

### 4-C. LDAP / Active Directory

**From the directory administrator**: the directory address, a service account for searching users
(its DN and password), and the OUs (base DNs) holding users and groups.

**The `LDAP / AD` tab**

| Field | Value (Active Directory example) | Notes |
| --- | --- | --- |
| `Enable LDAP / Active Directory` | on | |
| `Directory URL` | `ldaps://ad.example.com:636` | Use `ldaps://`, or turn on `StartTLS` below. Otherwise passwords cross the network in the clear |
| `Bind DN` | `cn=svc-ontul,ou=service,dc=example,dc=com` | The service account that finds user entries |
| `Bind password` | the service account's password | |
| `User base DN` | `ou=people,dc=example,dc=com` | |
| `User filter` | `(sAMAccountName={0})` | `{0}` is the login name. OpenLDAP uses the default `(uid={0})` |
| `Group base DN` | `ou=groups,dc=example,dc=com` | |
| `Group filter` | default `(member={0})` | `{0}` is the user's DN. The user's `memberOf` is read as well |
| `Group name attribute` | default `cn` | |
| `StartTLS` | on when using `ldap://` | |

Click **`Save`** → **`Test the bind`**. This checks that the service account can connect and bind.

There is no separate button for signing in: users type their directory account and password into the
**normal login form**. The local password is checked first; if it does not match, the directory is tried.

---

## 5. Group mapping: the `Identity mapping` tab

| Field | Value | Notes |
| --- | --- | --- |
| `Group mappings` | `DATA-ADMIN:ontul-admins,DATA-ANALYST:ontul-readers` | `idpGroup:ontulGroup` pairs, comma-separated. Names must match **exactly, including case** |
| `Allow identities whose groups map to nothing` | off (recommended) | When on, users whose groups map to nothing can sign in but can do nothing |
| `Federated session seconds` | default `3600` | How long an SSO session lasts. Shorter means a block at the IdP takes effect sooner |

- Left empty, IdP group names are used as Ontul group names unchanged.
- **Once set, any group not listed is dropped.** Creating a group at the IdP does not grant access by itself.
- A leading `/` on group names, as Keycloak sends them (`/DATA-ADMIN`), is stripped before matching.
- The Ontul groups on the right must be groups **actually created in section 2**. Mapping to a group
  that does not exist lets the user sign in, but with no policies every action is denied.

Click **`Save`**.

---

## 6. Verify the login

1. In another browser (or a private window), open `https://iam.example.com/admin/`.
2. Check that the login page shows **`Sign in with OpenID Connect`** / **`Sign in with SAML`** below `or`.
   With only LDAP enabled, the form shows the hint "Directory accounts sign in with the form above".
3. Click the button, sign in at the IdP, and land back in the console — that is success.
4. A user mapped to the administrator group sees `Save` enabled on the `Single Sign-On` screen; a user in
   the read-only group sees the `Read-only` notice.

**On the first integration, keep the administrator window (signed in as local `admin`) open** and test
in another window. Local login keeps working regardless of SSO.

### Checking from the API (optional)

```bash
# Which providers the login page offers (no authentication needed)
curl -s https://iam.example.com/admin/auth/sso/providers
# {"providers":["oidc","saml"],"localLoginEnabled":true}

# A token for an LDAP account (for scripts and SDKs)
curl -s -X POST https://iam.example.com/v1/api/auth/sso \
  -H 'Content-Type: application/json' \
  -d '{"username":"jdoe","password":"<directory password>"}'
# {"accessToken":"...","username":"jdoe","federated":true,"idp":"LDAP","groups":["ontul-readers"]}
```

---

## 7. Troubleshooting

When a login fails, the reason is shown above the login form. The master log records the same
(`Console SSO refused`, `Federated login refused`, `Directory login refused`).

| Symptom | Cause and fix |
| --- | --- |
| A `redirect_uri` error at the IdP | `Redirect URI` differs from what the IdP has registered. Match the scheme (`http`/`https`), the port and any trailing `/` |
| `invalid_scope` | `Scopes` contains a scope the IdP does not know, such as `groups`. Empty it |
| After OIDC/SAML login: "Signed in as …, but no group from the identity provider maps to a group on this cluster" | The IdP login succeeded and **the group mapping came out empty.** Check ① that the IdP includes the groups claim/attribute, ② that `Groups claim` / `Groups attribute` names it correctly, ③ that the left-hand names in `Group mappings` match the IdP group names, including case |
| LDAP: "Invalid credentials" on the login form | The login form **does not distinguish** a wrong password from an empty group mapping. Use the command below: 401 is an account, password or search problem; **403** means the password was right and the group mapping is empty (no password reset needed) |
| Signed in, but every screen is a permission error | The mapped Ontul group does not exist or has no policy (section 2) |
| Signed in, but the settings screens are read-only | The user's groups lack `AdministratorAccess` |
| SAML: "The assertion consumer received no SAMLResponse" | The IdP did not POST to the ACS. Check that the IdP's ACS binding is HTTP-POST and that the ACS address is right |
| SAML: signature verification fails | The IdP's signing certificate changed. Fetch the IdP metadata again and `Read the provider's metadata` |
| SAML: the IdP rejects the request | Empty `NameID format`. If the IdP requires signed requests, see step 3 of 4-B |
| LDAP: `Test the bind` fails | Check the address, port, certificate (for ldaps, the master JVM must trust the directory's certificate), Bind DN and password |
| LDAP: the command below answers 401 | User search failed (`User base DN` / `User filter`) or the password is wrong. For AD, check it is `(sAMAccountName={0})` |
| Only some attempts fail | `Redirect URI` / ACS points at one master. Use the proxy's address (1-2) |

Telling LDAP login failures apart:

```bash
curl -s -w '\n%{http_code}\n' -X POST https://iam.example.com/admin/auth/sso/ldap/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"jdoe","password":"<directory password>"}'
# 200 → fine, 401 → account / password / search settings, 403 → no group mapping
```

---

## 8. Operating notes

- **Leavers and revocation**: blocking a user at the IdP stops new logins immediately. A session already
  issued lasts until `Federated session seconds` runs out — Ontul is not told about changes at the IdP.
  An SSO session has no refresh token, so it is never extended.
- **Engine authorization and SSO**: the authorization plugins of the query, batch and streaming engines
  look users up **by name**. An SSO user's groups are live only for the session length after that user
  signs in through SSO. Outside a session the name resolves to no groups and is denied. Create service
  accounts and batch users that use the engines continuously as **local users** and put them in groups.
- **A local user of the same name**: signing in through SSO still applies the local account's groups and policies.
- The settings are replicated in the cluster store, so adding or restarting masters needs no re-entry.
- Only administrators can change SSO. Other users can view this screen and download the SP metadata,
  but cannot save, test, generate keys or import metadata.

## See also

- [Single Sign-On (OIDC/SAML/LDAP)](sso.md) — how it works, and the `ontul.properties` properties
- [IAM](iam.md) — groups and policies
- [IAM Policy Templates](iam-policy-templates.md) — policies to attach to a group

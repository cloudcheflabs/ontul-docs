# Single Sign-On (OIDC, SAML, LDAP)

Ontul can hand authentication to an external identity provider, so the cluster
stops being another place that holds passwords and becomes another thing your
directory governs. Three providers are supported — OpenID Connect, SAML 2.0, and
LDAP / Active Directory — and all three work on both surfaces.

## SSO means two different things here

This is the part most integrations get wrong, so it is worth being precise before
any configuration.

**The admin console is a browser.** It can be redirected, so it uses the flows
built for that: OIDC Authorization Code with PKCE, or SAML 2.0 Web Browser SSO.

**The REST surface is not a browser and cannot redirect anywhere.** An SDK, a
script or a scheduler has nowhere to display a login page. So it posts the
credential it already holds and receives an ordinary Ontul bearer token:

```bash
# an OIDC ID token the application already obtained
curl -X POST http://ontul:8080/v1/api/auth/sso \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","idToken":"eyJhbGciOi..."}'

# or a base64 SAML assertion
curl -X POST http://ontul:8080/v1/api/auth/sso \
  -d '{"username":"alice","samlResponse":"PHNhbWxwOl..."}'

# or directory credentials, which is the simplest case of all
curl -X POST http://ontul:8080/v1/api/auth/sso \
  -d '{"username":"alice","password":"<directory password>"}'

# → {"accessToken":"...","username":"alice","federated":true,"idp":"OIDC","groups":["analysts"]}
```

That is the same token `POST /v1/api/auth/accesskey` returns, so **nothing
downstream changes** — the SDK, the JDBC driver over Arrow Flight SQL, the MCP
server and every `/v1/api/*` endpoint accept it as they always did.

Directory credentials also work on the ordinary console login form. An operator
whose account lives in LDAP or Active Directory types their normal password into
the normal form, which is the point of supporting that provider: there is nothing
extra to learn.

## What decides permissions

Nothing about the provider does. A federated identity arrives carrying group
names; those are mapped to groups **on this cluster**, and the policies attached
to those groups authorize every request.

This matters more in Ontul than in a product with one plane, because of
`/v1/api/authz/check` — the call the Trino, Spark and Flink plugins make before
every statement. It resolves a caller **by name**, and it never sees a token. So a
federated login records a short-lived session that carries its mapped groups, and
policy resolution reads it. The consequence worth knowing:

- Every authorization path honours SSO, including column masking, column deny and
  row filters, with no engine-side configuration.
- **No local user account is created.** A federated caller has no password and
  nothing to persist; your directory is the record. Creating an account for each
  person who ever signed in would put them all in the IAM list and in the
  replicated snapshot, and removing them from the directory would leave the copy
  behind still authorizing them.
- A session lasts `ontul.sso.federated.session.seconds` (default 3600). After
  that the identity resolves to nothing until they sign in again.

!!! note "A local account of the same name wins"
    If a stored user and a federated identity share a name, the stored account's
    policies apply. Creating a local user is a deliberate act by an administrator,
    so it is the more specific statement of intent — and resolving the other way
    round would let whoever controls the directory decide what a local account
    can do.

### Group mapping

```properties
ontul.sso.group.mappings=db-admins:ontul-admins,analysts:ontul-readers
```

Left empty, provider group names are used as they are — the common case where the
directory already uses this product's group names. **Once set, the mapping is
exhaustive:** a group not named in it is dropped, so creating a group at the
provider cannot grant access here by itself.

An identity whose groups all map to nothing is refused, not admitted with no
groups. Such a session has no policies and is denied every action, so letting it
in produces someone who is signed in and can do nothing, left to work out why
from permission errors. Set `ontul.sso.allow.unmapped.groups=true` if you would
rather allow it.

The directory login endpoint reports that case as **403**, separately from a wrong
password's 401 — otherwise an operator goes and resets a password that was right.

## Configuring it

Everything below can be set in the **admin console under Single Sign-On**, which
stores it in the replicated metadata store and applies it to every master — no
file edits, no restart. The properties file is still read for anything left unset,
so a cluster configured by file keeps working untouched.

Local passwords keep working while SSO is on. Enabling it cannot lock you out.

### OpenID Connect

```properties
ontul.sso.oidc.enabled=true
ontul.sso.oidc.issuer=https://keycloak.example.com/realms/company
ontul.sso.oidc.client.id=ontul-console
ontul.sso.oidc.client.secret=…
ontul.sso.oidc.redirect.uri=https://ontul.example.com/admin/auth/sso/oidc/callback
ontul.sso.oidc.groups.claim=groups
```

Endpoints are read from the issuer's discovery document, so they are not
configured individually. The ID token's signature is verified against the
provider's published key set, and its issuer, audience and expiry are all checked
— a token issued for a different application is refused even though it is genuine
and correctly signed.

!!! note "The `groups` scope"
    `ontul.sso.oidc.scopes` deliberately does **not** include `groups`. It is not a
    standard scope, and a provider that does not define it rejects the whole
    authorization request with `invalid_scope` — so asking for it by default logs
    nobody in. Group membership comes from a claim the provider is configured to
    include. Add a scope here only if your provider documents one.

### SAML 2.0

A SAML integration is an exchange of metadata, not a form-filling exercise.

1. **Download this cluster's metadata** from the console (or
   `GET /admin/sso/saml/metadata`) and give it to whoever administers your
   identity provider. It carries the entity ID, the assertion consumer URL and,
   if a keypair has been generated, the certificate.
2. **Paste your provider's metadata** into the console. The entity ID, sign-on URL
   and signing certificate are read from it — which beats transcribing three
   fields by hand, where the typos are.

```properties
ontul.sso.saml.enabled=true
ontul.sso.saml.idp.entity.id=https://idp.example.com/realms/company
ontul.sso.saml.idp.sso.url=https://idp.example.com/protocol/saml
ontul.sso.saml.idp.certificate=MIIC…
ontul.sso.saml.sp.entity.id=ontul
ontul.sso.saml.sp.acs.url=https://ontul.example.com/admin/auth/sso/saml/acs
```

Every assertion is checked four ways, and each one is a real attack if skipped:

| Check | What it prevents |
| --- | --- |
| Signature, against the provider's certificate | An assertion the caller wrote |
| Audience | A genuine assertion for another service logging in here |
| Validity window | A captured assertion replayed forever |
| Issuer | Any provider the caller can reach being trusted |

On top of those, an assertion already used is refused. That record is kept **in
ZooKeeper, not in one master's memory**: an assertion is a bearer document, and a
replay arrives at whichever master the load balancer picks — usually not the one
that saw the original.

**Encrypted assertions and signed requests.** Several providers encrypt assertions
or require the authentication request to be signed. Both need a service-provider
keypair — generate one from the console, then re-import the SP metadata at your
provider so it picks up the new certificate. An encrypted assertion must carry its
own signature: encryption proves who the assertion was *for*, never who wrote it.

**NameID format** is left empty by default, which omits the request entirely and
lets the provider issue whatever it is configured for. Naming one breaks more
integrations than it fixes — several providers refuse a request asking for a
format they do not issue.

### LDAP / Active Directory

```properties
ontul.sso.ldap.enabled=true
ontul.sso.ldap.url=ldaps://ad.example.com:636
ontul.sso.ldap.bind.dn=cn=svc-ontul,ou=service,dc=example,dc=com
ontul.sso.ldap.bind.password=…
ontul.sso.ldap.user.base.dn=ou=people,dc=example,dc=com
ontul.sso.ldap.user.filter=(sAMAccountName={0})
ontul.sso.ldap.group.base.dn=ou=groups,dc=example,dc=com
ontul.sso.ldap.group.filter=(member={0})
```

Authentication is **search then bind**. A service account finds the user's entry —
their DN is something the product cannot construct, since Active Directory puts
people under `CN=John Doe,OU=Staff,…` where neither component is the login name —
and the password is then checked by binding as that DN.

That second bind *is* the authentication. Reading a password attribute and
comparing it would be wrong even where the directory allows it: only the server
knows how its own hashes are salted, and account lockout, expiry and disabled
flags are enforced on bind and nowhere else.

Group membership is read **both ways**: from the user's `memberOf` and from a
search of the group tree. Directories disagree about which side records it —
OpenLDAP usually keeps it on the group, Active Directory mirrors it onto the user
— and reading only one way silently returns no groups against half the servers in
the field.

!!! warning "Use TLS"
    Without `ldaps://` or `ontul.sso.ldap.starttls=true`, the bind password crosses
    the network in the clear.

## Behind a load balancer

Both browser flows work on any master, regardless of which one started them. The
login state — including the PKCE verifier — is sealed with a key every master
derives from `ONTUL_MASTER_KEY` and carried in the `state` parameter itself rather
than held in memory on one master. Unsealing it is also what proves this cluster
issued it, which is the login-CSRF check.

Federated sessions replicate with the IAM snapshot, so a token minted on one
master is honoured on all of them.

Neither is a detail: without them SSO works on a single master and fails on
roughly half of all attempts in a cluster — and it fails looking like a problem at
the identity provider.

## Revocation

Disabling someone at the provider stops new logins immediately. Sessions already
issued keep working until they expire — this cluster is not told about the change.
`ontul.sso.federated.session.seconds` bounds how long that gap lasts; shorter is
safer.

A federated session gets no refresh token for the same reason: renewing would keep
someone signed in after the directory disabled them.

## Password storage

Independent of SSO, local passwords are stored as PBKDF2-HMAC-SHA256 hashes.
Values written under the previous unsalted SHA-256 still verify and are rewritten
on the owner's next successful login, which is the only moment the plaintext is
available to hash — nobody is locked out by the change. The iteration count travels
with each stored hash, so raising
`ontul.auth.password.hash.iterations` (default 600000) does not invalidate
existing passwords.

## See also

- [Identity and Access Management](iam.md) — the groups and policies a federated identity maps onto.
- [IAM Policy Templates](iam-policy-templates.md) — what to attach to a mapped group.

# Sign-in with OIDC (single sign-on)

NodeGuard can connect to any OpenID Connect provider, for example **Authentik**, **Keycloak**, **Authelia**, **Zitadel**, **Pocket ID** or **Kanidm**. The login page then shows an additional **Sign in with …** button. Sign-in with a local password stays available.

NodeGuard uses the authorization code flow with PKCE. The provider data (endpoints, keys) is loaded automatically from `<issuer>/.well-known/openid-configuration`.

German version: [oidc.md](oidc.md)

## 1. Create a client at the identity provider

Create a new OIDC client (application) at the provider:

| Setting | Value |
| --- | --- |
| Client type | *Confidential* (with client secret), or *Public* without a secret, PKCE only |
| Grant type | Authorization code |
| Redirect URI | `http(s)://<server-address>/api/auth/oidc/callback` |
| Scopes | `openid profile email`, plus `groups` for group membership (depending on the provider) |

NodeGuard shows the exact redirect URI during the initial setup and under **Settings → Sign-in** (with a copy button). It must match exactly, including `http`/`https` and the port.

Notes on individual providers:

- **Authentik:** provider type *OAuth2/OpenID Provider*. The issuer is `https://<authentik>/application/o/<slug>/`. Groups are included in the `groups` claim with the `profile` scope.
- **Keycloak:** enable *Client authentication* for the client. The issuer is `https://<keycloak>/realms/<realm>`. For groups, create a *Group Membership* mapper with claim name `groups`, or use `realm_access.roles` (realm roles) as the groups claim.
- **Authelia:** define the client under `identity_providers.oidc.clients` with `redirect_uris` as above. The issuer is the Authelia address. For groups, add the `groups` scope.

## 2. Set up NodeGuard

### During the initial setup

1. Open the web app. The **initial setup** appears automatically.
2. Switch to the **Single sign-on** tab.
3. Enter the **Issuer URL**, **Client ID** and, if applicable, **Client secret**. Under *Allowed groups* you can define which groups may sign in.
4. Click **Save and sign in at the provider** and sign in at the provider.

The account you sign in with becomes the NodeGuard administrator. It has no local password. NodeGuard checks the provider address when you save. If it is not reachable, the error appears directly in the form.

You can also create a local account as before and set up OIDC later.

### Later, in the settings

Under **Settings → Sign-in** you can change all values and turn OIDC on or off. There you can also link your existing account (see below).

### Using environment variables

Alternatively, OIDC can be configured entirely with environment variables. As soon as `OIDC_ISSUER` is set, only these values apply, and the forms in the web app show them read-only.

```yaml
  nodeguard:
    environment:
      OIDC_ISSUER: "https://auth.example.com/application/o/nodeguard/"
      OIDC_CLIENT_ID: "nodeguard"
      OIDC_CLIENT_SECRET: "…"
      OIDC_DISPLAY_NAME: "Authentik"
      OIDC_ALLOWED_GROUPS: "nodeguard-admins"
```

| Variable | Default | Meaning |
| --- | --- | --- |
| `OIDC_ISSUER` | – | Issuer URL of the provider. Enables configuration via environment. |
| `OIDC_CLIENT_ID` | – | Client ID. |
| `OIDC_CLIENT_SECRET` | empty | Client secret. Empty = public client (PKCE only). |
| `OIDC_ENABLED` | `true` | `false` hides OIDC sign-in without removing the values. |
| `OIDC_DISPLAY_NAME` | `Single sign-on` | Label of the button: "Sign in with …". |
| `OIDC_SCOPES` | `openid profile email` | Requested scopes, must contain `openid`. |
| `OIDC_USERNAME_CLAIM` | `preferred_username` | Claim used as username for new accounts. |
| `OIDC_GROUPS_CLAIM` | `groups` | Claim with the groups, nested paths such as `realm_access.roles` are supported. |
| `OIDC_ALLOWED_GROUPS` | empty | Comma-separated groups. Empty = every account of the provider may sign in. |
| `OIDC_AUTO_CREATE_USERS` | `false` | Unknown identities get their own account on first sign-in. |
| `OIDC_LINK_BY_USERNAME` | `false` | Unknown identities are linked to the local account with the same name. |
| `OIDC_REDIRECT_URI` | automatic | Only needed if NodeGuard cannot detect its public address behind a reverse proxy. |

## 3. Mapping identities to NodeGuard accounts

An identity at the provider is uniquely assigned to a NodeGuard account by the pair *issuer + subject (`sub`)*. The username or e-mail address alone is not enough to sign in.

The mapping is created in one of these ways:

1. **Initial setup via OIDC:** the first account is created from the identity.
2. **Linking in the settings:** signed in with your local account, click **Settings → Sign-in → Link identity** and sign in at the provider. Afterwards both password and single sign-on work. Links can also be removed there.
3. **Create accounts automatically** (*Create accounts automatically*): every allowed identity gets its own account on first sign-in. If you use this, restrict access with *Allowed groups*.
4. **Link by username** (*Link by username*): an unknown identity is linked to the local account with the same name. Only enable this if users cannot choose their own username at the provider.

Without a matching mapping, NodeGuard rejects the sign-in with a message. All NodeGuard accounts have the same (full) rights.

*Allowed groups* applies to every OIDC sign-in, including already linked accounts. Anyone removed from the group can no longer sign in via OIDC.

## 4. Session and sign-out

After signing in, the same session as with password login applies (encrypted cookie, 7 days). **Sign out** only ends the NodeGuard session, not the session at the provider. All sign-ins, links and changes to the settings are recorded in the **Audit log**.

## 5. Troubleshooting

| Message / symptom | Cause and solution |
| --- | --- |
| `Could not load the provider configuration from …` when saving | Issuer URL is wrong or not reachable from the NodeGuard container. Check with `docker compose exec nodeguard wget -qO- <issuer>/.well-known/openid-configuration`. |
| Provider reports `invalid redirect_uri` | The redirect URI at the provider does not match the one shown in NodeGuard exactly. Behind a reverse proxy, set `X-Forwarded-Proto`/`X-Forwarded-Host` or enter the redirect URI explicitly. |
| `No NodeGuard account is linked to this identity.` | The identity is not assigned to an account. Sign in with your password and link it under **Settings → Sign-in**, or enable *Create accounts automatically*. |
| `Your account is not in a group that may use NodeGuard.` | Groups are missing from the token. Add the `groups` scope or set up a group mapper at the provider and check the groups claim. |
| `The sign-in request expired or was started in another browser.` | More than 10 minutes passed between start and return, or cookies are blocked. Try again; over plain HTTP you may need `SESSION_COOKIE_SECURE=false`. |
| `Single sign-on failed.` | Details are in the container log (`docker compose logs nodeguard`, entries with `[OIDC]`), for example a wrong client secret. |
| Provider with its own or self-signed certificate | Make the CA known to the container: mount the certificate and set `NODE_EXTRA_CA_CERTS=/path/to/ca.pem`. Providers over plain HTTP are supported as well. |
| Provider down, sign-in not possible | Accounts with a password can still sign in locally. As a last resort run `docker compose exec nodeguard reset-password` and repeat the initial setup; the OIDC settings are kept. |

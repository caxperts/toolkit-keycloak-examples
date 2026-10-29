# Keycloak Example Setup

This document shows how to set up Keycloak for the Toolkit with the template files in the folder [`templates`](templates/). It also lists the manual steps that the templates cannot do.

## What the Toolkit Needs in Keycloak

| Object | Purpose | Toolkit setting |
|---|---|---|
| Client `connector` | Login in the web browser. Public client without refresh tokens. | `KEYCLOAK_ID` / `Keycloak__ClientId` |
| Client `toolkit-upv` (optional) | Login of the viewer inside UDiTH App and browser-based viewing. Public client with refresh tokens. | `Upv__ClientId` |
| Client `serviceclient` | Synchronization of users and groups by the background worker. Confidential client with a service account. | `SERVICE_CLIENT_ID` / `Keycloak__AdminClientId`, `SERVICE_CLIENT_SECRET` / `Keycloak__AdminClientSecret` |
| `groups` claim | Puts the group names of the user into the token. The Toolkit uses them for permissions. | — |
| Group `Admin` | Members are Toolkit administrators. | — |
| Identity provider `entra` (optional) | Login with Microsoft Entra ID. | — |

The client `toolkit-upv` is optional:

- Without `toolkit-upv`, keep `Upv__ClientId` empty. The Toolkit then uses the browser login with `connector` inside UDiTH App too.
- With `toolkit-upv`, the viewer signs in with its own login. The viewer renews its login with a refresh token. The browser client `connector` issues no refresh tokens, for security reasons. Thus the viewer login needs its own client. Do not set `Upv__ClientId=connector`, because then UDiTH App opens a new browser login every few minutes.
- Behind an authentication proxy such as Azure EasyAuth, set `USE_BROWSER_AUTH_IN_UPV=true`. The Toolkit then ignores `Upv__ClientId`, and you do not need `toolkit-upv`.

The name `toolkit-upv` is only the default of the template. You can use a different name.

## Template Files

| File | Content | Use |
|---|---|---|
| [`toolkit-realm.json`](templates/toolkit-realm.json) | A complete new realm: all clients, the client scope `groups`, the group `Admin`, the roles of the service account, the Entra ID login, and the token and session settings. | New realm |
| [`toolkit-connector-client.json`](templates/toolkit-connector-client.json) | Client `connector`, with its own `groups` mapper. | Existing realm |
| [`toolkit-upv-client.json`](templates/toolkit-upv-client.json) | Client `toolkit-upv`, with its own `groups` mapper. | Existing realm (optional) |
| [`toolkit-service-client.json`](templates/toolkit-service-client.json) | Client `serviceclient`. | Existing realm |
| [`groups-client-scope.json`](templates/groups-client-scope.json) | Client scope `groups`. | Existing realm (optional) |
| [`entra-oidc-identity-provider.json`](templates/entra-oidc-identity-provider.json) | Identity provider `entra` for Microsoft Entra ID. | Existing realm (optional) |

The files contain no secrets. Keycloak creates the client secret of `serviceclient`.

## Step 1: Change the Values in the Files

Before you import a file, replace these example values:

| Example value | Replace with | Files |
|---|---|---|
| `https://toolkit.example.com` | The public URL of the Toolkit (`KEYCLOAK_REDIRECT_URL`). With path-based multi-tenancy, include the instance path, and add the redirect URIs once for each instance. | Client files, realm file |
| `https://webservices.example.com/UpvWebServices` | The URL of the UPV Web Services (`REDIRECT_WEBSERVICES_URL`). | `toolkit-upv-client.json`, realm file |
| `https://webservices.example.com` | The origin of the UPV Web Services (scheme and host only). | `toolkit-upv-client.json`, realm file |
| `account` in `included.client.audience` | The value of `Keycloak__Audience`, if it is not `account`. | Client files, realm file |
| `toolkit` in `"realm"` | The name of your realm. | Realm file |
| `00000000-0000-0000-0000-000000000000` | Your Entra tenant ID (five times in each file). | `entra-oidc-identity-provider.json`, realm file |
| `11111111-1111-1111-1111-111111111111` | The application (client) ID of your Entra app registration. | `entra-oidc-identity-provider.json`, realm file |

- If you do not use browser-based viewing, remove the `loginPopup.html` redirect URI and the web origin from `toolkit-upv`.
- If you do not use Entra ID, remove the `identityProviders` section from the realm file.

## Step 2a: New Realm

1. In Keycloak, click **Create realm**.
2. Select `toolkit-realm.json` as **Resource file**, then click **Create**.
3. Go to **Clients** → `serviceclient` → **Credentials**. Copy the client secret to `SERVICE_CLIENT_SECRET`.
4. If you use Entra ID, do the steps in [Step 3: Login with Microsoft Entra ID](#step-3-login-with-microsoft-entra-id). The realm already contains the identity provider.

## Step 2b: Existing Realm

### Clients

For each client file that you need:

1. Go to **Realm settings** → **Action** → **Partial import**.
2. Select the file.
3. Select the **Clients** check box.
4. Keep **If a resource exists** at **Fail**, then click **Import**.

Keycloak does not change a client that exists. Compare an existing client with the template manually.

Use **Partial import**, not **Clients** → **Import client**. The files have the partial import format (`{ "clients": [ … ] }`).

### Service Account Roles of `serviceclient`

1. Go to **Clients** → `serviceclient` → **Service accounts roles**.
2. Click **Assign role** → **Client roles**.
3. Assign the `realm-management` roles `view-users` and `query-groups`.
4. Go to **Credentials**. Copy the client secret to `SERVICE_CLIENT_SECRET`.

The worker only reads users and their groups. Do not give it more roles. The Toolkit does not need the roles `manage-users` or `view-realm`, and the service account user needs no group. If an existing realm gives these to `serviceclient`, you can remove them.

### Group `Admin`

If the realm has no group `Admin`, go to **Groups** → **Create group**. Enter the name `Admin`, then click **Create**.

### Client Scope `groups` (Optional)

The client files contain their own `groups` mapper, because partial import cannot create client scopes. If you prefer one shared client scope, create it **before** you import the clients:

- With the Keycloak admin CLI:
  ```bash
  kcadm.sh create client-scopes -r <realm> -f groups-client-scope.json
  ```
- Or in the admin console:
  1. Go to **Client scopes** → **Create client scope**. Enter the name `groups`, then click **Save**.
  2. Go to **Mappers** → **Configure a new mapper** → **Group Membership**.
  3. Enter the name `groups` and the token claim name `groups`. Set **Full group path** to off, then click **Save**.

Then add the scope to each client: go to **Clients** → client → **Client scopes** → **Add client scope**. Select `groups`, then click **Add** → **Default**. Remove the `groups` mapper of the client (**Client scopes** → `<client>-dedicated`), or keep it: both give the same group names.

## Step 3: Login with Microsoft Entra ID

This step is optional.

1. In an existing realm, import `entra-oidc-identity-provider.json`: go to **Realm settings** → **Action** → **Partial import**. Select the file. Select the **Identity providers** check box, then click **Import**. A new realm from `toolkit-realm.json` already has the identity provider.
2. In the Entra app registration, add the platform **Web** with this redirect URI:
   ```
   https://<keycloak>/realms/<realm>/broker/entra/endpoint
   ```
3. In Keycloak, go to **Realm settings** → **Keys**. In the row **RS256**, click **Certificate** and copy it. Save it as a `.cer` file.
4. In the Entra app registration, go to **Certificates & secrets** → **Certificates**. Upload the `.cer` file.

Keycloak signs its requests to Entra with the realm key (`private_key_jwt`). Thus you do not need a client secret. When the realm key changes, upload the new certificate.

Entra finds the certificate through the X.509 headers of the signed request. Thus the identity provider must have **Add X.509 Headers to the JWT** set to on. The template files set it.

### Administrators from Entra ID

The identity provider gives users no Toolkit groups.

> **Warning:** Do not add a mapper that puts all users into the group `Admin` (for example a **Hardcoded Group** mapper). Then every user of your Entra tenant is a Toolkit administrator.

To make selected users administrators:

1. In the Entra app registration, create an app role, for example `Toolkit.Admin`. Assign it to the users.
2. In Keycloak, open the identity provider → **Mappers** → **Add mapper**. Select **Advanced Claim to Group**.
3. Add the claim `roles` with the value `Toolkit.Admin`. Select the group `/Admin`, then click **Save**.

## Step 4: Configure the Toolkit

Set these values on the Toolkit container, or under System Settings:

| Setting | Value |
|---|---|
| `KEYCLOAK_ISSUER` | `https://<keycloak>/realms/<realm>` |
| `KEYCLOAK_ID` | `connector` |
| `KEYCLOAK_REDIRECT_URL` | The public URL of the Toolkit |
| `SERVICE_CLIENT_ID` | `serviceclient` |
| `SERVICE_CLIENT_SECRET` | The client secret from Keycloak |
| `Upv__ClientId` | `toolkit-upv` (only if you use the client `toolkit-upv`) |
| `REDIRECT_WEBSERVICES_URL` | The URL of the UPV Web Services (only for browser-based viewing) |

If `Upv__ClientId` is not set, the Toolkit uses the browser login inside UDiTH App, the same as `USE_BROWSER_AUTH_IN_UPV=true`.

## Step 5: Check the Setup

1. Go to **Clients** → `connector` → **Client scopes** → **Evaluate**.
2. Select a user who is a member of `Admin`.
3. Open **Generated access token**. Make sure that the token contains:
   - `"aud"` with `account` (or your `Keycloak__Audience`),
   - `"groups"` with `Admin` (not `/Admin`).
4. If you use `toolkit-upv`, do the same check for `toolkit-upv`.
5. If you use `toolkit-upv`, open the Toolkit in UDiTH App. After the first login, no new browser login must open every few minutes. The UPV log shows "Renewing access token using refresh token".

## Test Realms

The end-to-end tests get tokens with the password grant of the `connector` client. In a realm for these tests, set **Direct access grants** to on for `connector`. Keep it off in production realms.

## Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| In **Partial import**, the **Import** button stays disabled. | No check box is selected, or the file is not in partial import format. | Select the **Clients** or **Identity providers** check box. Use the template files without changes to their structure. |
| Import fails with "String or binary data would be truncated … column 'DESCRIPTION'". | A client description is longer than 255 characters. | Make the description shorter. |
| Keycloak log: "Referenced client scope 'groups' doesn't exist. Ignoring". | The client references the client scope `groups`, but the realm does not have it. | Create the client scope first, or use the client files from the folder `templates`. They contain their own `groups` mapper. |
| Administrators get "403 Forbidden" in the settings. | The token has no `groups` claim, or the claim contains `/Admin`. | Do [Step 5](#step-5-check-the-setup). Set **Full group path** to off. |
| Login with Entra ID shows "Unexpected error when authenticating with identity provider". | Keycloak sends the signed request without X.509 headers. Entra cannot find the certificate. | Go to **Identity providers** → `entra`. Set **Add X.509 Headers to the JWT** to on, then click **Save**. Reload the login page. |
| UDiTH App opens a new browser login every few minutes. | UDiTH App uses a client without refresh tokens. | Import `toolkit-upv` and set `Upv__ClientId=toolkit-upv`. Or make `Upv__ClientId` empty to use the browser login. |
| A new realm has no client scopes `profile`, `email` or `roles`. | The realm file defines its own client scope without the attribute `CreateDefaultClientScopes`. | Keep `"CreateDefaultClientScopes": "true"` in the realm file. |

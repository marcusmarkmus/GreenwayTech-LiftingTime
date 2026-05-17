# GreenwayTech LiftingTime

## What is this?
Lifting time is meant to be a lifting planner/diary/notebook, useable from mobile and desktop.  
Core features are:  
* Create re-usable workout sessions (i.e. "Chestday", "Full body session"), with the built in exercise selections, or create your own exercises.
* Create work out programs with the sessions.
* Easily track your sessions with weights, reps, and optionally Reps in Reserve and/or Relative Effort.   
* Rest-timer and workout timer.
* Statistics per muscle, exercise, workouts.

## How do I selfhost this?
Todo: docker compose for Auth, db, webapp and webapi.   
OIDC: how to bring your own.  
Maybe something to put in wiki.

## How do I contribute to the project?
todo: Road to F5, guidelines, etc. 
also candidate for the wiki.


## TODO: incorporate this in the wiki for selfhosting
## OIDC admin roles

LiftingTime uses OIDC/JWT for authentication and a single application role for admin access:

```text
liftingtime.admin
```

The API only grants access to `/admin/*` endpoints when the authenticated user's **access token** contains that exact role value in the configured role claim. The default role claim is:

```json
{
  "Oidc": {
    "RoleClaimType": "role"
  }
}
```

With the default configuration, this access token payload grants admin access:

```json
{
  "sub": "user-id",
  "role": "liftingtime.admin"
}
```

These payloads do **not** grant admin access unless `RoleClaimType` is changed:

```json
{
  "sub": "user-id",
  "groups": ["liftingtime.admin"]
}
```

```json
{
  "sub": "user-id",
  "entitlements": ["liftingtime.admin"]
}
```

This strict behavior is intentional. LiftingTime does not treat every group or entitlement as an admin role by default, because different providers use those claims for different meanings. Only the configured claim type is used for admin authorization.

### Access token vs. ID token/userinfo

The Blazor web app signs users in with OIDC and stores a local cookie. The API does not use that cookie. API calls are sent with a bearer **access token**, and the API authorizes admin endpoints from claims in that access token.

If the admin page is visible but the admin panel data fails with `403 Forbidden`, the usual cause is:

```text
The web app can see the admin role, but the access token sent to the API does not contain it.
```

Fix this in your OIDC provider by adding the admin role claim to the **access token**, not only to the ID token or userinfo response.

### Common configurations

Default flat `role` claim:

```json
{
  "Oidc": {
    "RoleClaimType": "role"
  }
}
```

Access token:

```json
{
  "role": "liftingtime.admin"
}
```

Multiple roles in the same `role` claim also work:

```json
{
  "role": ["liftingtime.user", "liftingtime.admin"]
}
```

Provider uses `roles`:

```json
{
  "Oidc": {
    "RoleClaimType": "roles"
  }
}
```

Access token:

```json
{
  "roles": ["liftingtime.admin"]
}
```

Provider uses `groups`:

```json
{
  "Oidc": {
    "RoleClaimType": "groups"
  }
}
```

Access token:

```json
{
  "groups": ["liftingtime.admin"]
}
```

Provider uses `entitlements`:

```json
{
  "Oidc": {
    "RoleClaimType": "entitlements"
  }
}
```

Access token:

```json
{
  "entitlements": ["liftingtime.admin"]
}
```

Docker/environment variable form:

```bash
Oidc__RoleClaimType=groups
```

### Provider notes

Authentik commonly uses groups or entitlements. If your access token contains:

```json
{
  "groups": ["liftingtime.admin"]
}
```

set:

```bash
Oidc__RoleClaimType=groups
```

If it contains:

```json
{
  "entitlements": ["liftingtime.admin"]
}
```

set:

```bash
Oidc__RoleClaimType=entitlements
```

Keycloak often stores roles in nested claims such as:

```json
{
  "realm_access": {
    "roles": ["liftingtime.admin"]
  }
}
```

LiftingTime does not currently read nested role paths like `realm_access.roles`. Add a Keycloak protocol mapper that emits a flat claim in the access token instead:

```json
{
  "roles": ["liftingtime.admin"]
}
```

and configure:

```bash
Oidc__RoleClaimType=roles
```

Microsoft Entra ID app roles commonly use:

```json
{
  "roles": ["liftingtime.admin"]
}
```

Configure:

```bash
Oidc__RoleClaimType=roles
```

### Single-user self-hosting

If you are the only user and do not want to configure roles in your OIDC provider, you can explicitly grant admin access to every authenticated user:

```json
{
  "Oidc": {
    "GrantAdminToAllUsers": true
  }
}
```

Environment variable form:

```bash
Oidc__GrantAdminToAllUsers=true
```

Only use this when every authenticated user should be an administrator. In a multi-user deployment, configure real role claims instead.

### Troubleshooting

`401 Unauthorized` from the API means the access token is missing, expired, invalid, has the wrong audience, or was not issued by the configured authority.

`403 Forbidden` from `/admin/*` means the access token is valid, but admin authorization failed. Check that:

1. The role value is exactly `liftingtime.admin`.
2. The role is in the access token, not only the ID token or userinfo response.
3. `Oidc:RoleClaimType` matches the claim name in the access token.
4. Your provider emits a flat claim, for example `role`, `roles`, `groups`, or `entitlements`.
5. The application was restarted after changing OIDC configuration.

LiftingTime sets `MapInboundClaims = false` for both API bearer tokens and web app OIDC login. This keeps provider claim names unchanged. If your token contains `role`, configure `role`; if it contains `groups`, configure `groups`.
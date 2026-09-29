# External authentication built on ASP.NET Core authentication

Status: draft, for discussion. Proposal issue: TBD.

This document holds the full detail behind the proposal issue: current state, design decisions, and the split into small PRs. Nothing here is final. It is meant to be challenged.

## 1. Context

[jellyfin#17271](https://github.com/jellyfin/jellyfin/pull/17271) adds native OIDC (about 6000 lines, 46 files). The [team's position](https://github.com/jellyfin/jellyfin/pull/17271#issuecomment-5876899005) is:

1. first bring the authentication backend up to date and expose stable APIs for plugins,
2. then provide a login flow that every client can reuse,
3. with a feature proposal per step, then small targeted PRs.

Protocols (OIDC, SAML, ...) stay in plugins. The core only provides generic plumbing, built on the ASP.NET Core authentication framework rather than on custom abstractions.

### Prior work

| Ref | Content | State |
| --- | --- | --- |
| [jellyfin#14729](https://github.com/jellyfin/jellyfin/pull/14729) | Full auth refactor with custom abstractions (`IUserAuthenticationManager`, `AbstractAuthenticationProvider`, ...) and custom TOTP. | Closed |
| [jellyfin-meta#101](https://github.com/jellyfin/jellyfin-meta/issues/101) | Discussion of #14729. [@JPVenson](https://github.com/jellyfin/jellyfin-meta/issues/101#issuecomment-3245825963): no more custom implementations where ASP.NET Core already has a fitting solution; `AddAuthentication` is already used. | Closed |
| [jellyfin-meta#103](https://github.com/jellyfin/jellyfin-meta/issues/103) | Proxy header authentication. | Open |
| [jellyfin#17271](https://github.com/jellyfin/jellyfin/pull/17271) | Native OIDC, already based on `Microsoft.AspNetCore.Authentication.OpenIdConnect`. Too large for the core, but its ASP.NET Core part can be reused in a plugin. | Open |

Takeaway: a design built on custom abstractions will not be accepted. Each piece below states which ASP.NET Core mechanism it uses.

## 2. Current state

### Server (`jellyfin`, net10.0)

| Item | Where | Notes |
| --- | --- | --- |
| Provider interface | `MediaBrowser.Controller/Authentication/IAuthenticationProvider.cs` | `Authenticate(username, password)` only. No flow types, no client-facing metadata, no claims or groups in the result (`ProviderAuthenticationResult` = `Username` + `DisplayName`). |
| Extensions | same file | `IRequiresResolvedUser`, `IHasNewUserPolicy`, added over time. |
| Orchestration | `Jellyfin.Server.Implementations/Users/UserManager.cs` (`AuthenticateUser`, `AuthenticateLocalUser`) | Auth logic lives in `UserManager`: provider iteration, `AuthenticationProviderId` switching, lockout, implicit user creation. |
| User ↔ provider link | `User.AuthenticationProviderId` | One provider per user, no external identity (issuer/subject). Linked by username, so a rename on the IdP side breaks the link. |
| Groups | `Entities/Group.cs` | Entity exists (permissions, preferences) but is not used by the API or managers. |
| Tokens | `Entities/Security/Device.cs`, `ApiKey.cs` | Opaque token, no expiry, no refresh, stored in clear. One per device. |
| ASP.NET Core auth | `Jellyfin.Api/Auth/CustomAuthenticationHandler.cs`, `Jellyfin.Server/Extensions/ApiServiceCollectionExtensions.cs` | Already half used: `AddAuthentication()`, a `CustomAuthentication` handler producing a `ClaimsPrincipal`, authorization policies. This stays the default API scheme. |
| Plugin hook | `MediaBrowser.Controller/Plugins/IPluginServiceRegistrator.cs` | Plugins get the `IServiceCollection`, so they can already call `AddAuthentication().AddOpenIdConnect(...)`. No new ABI is needed to register a scheme. |
| Login endpoints | `Jellyfin.Api/Controllers/UserController.cs` | `AuthenticateByName`, `AuthenticateWithQuickConnect`, `ForgotPassword`. No anonymous discovery of login methods. |
| Existing `/Auth/*` | `SessionController.cs` (`GET /Auth/Providers`, `GET /Auth/PasswordResetProviders`), `ApiKeyController.cs` (`/Auth/Keys`) | `GET /Auth/Providers` already exists, admin only (`RequiresElevation`), listing `IAuthenticationProvider`s. It can't be reused for anonymous discovery without an API break. |
| QuickConnect | `Emby.Server.Implementations/QuickConnect/QuickConnectManager.cs`, `QuickConnectController.cs` | 6-digit code, 10 min TTL, in memory. Already a device-style flow, but approval needs a user already logged in elsewhere. |

### LDAP plugin (`jellyfin-plugin-ldapauth`)

The team pointed to it as the reference for how auth plugins should work.

- Implements `IAuthenticationProvider` and `IPasswordResetProvider`.
- Creates users itself (`userManager.CreateUserAsync`) and handles admin rights and library access through its own config.
- Has no frontend: it relies on the existing username/password form. That is what OIDC and SAML lack.

What it depends on today, which must not break:

| Dependency | Detail |
| --- | --- |
| Provider ID | `User.AuthenticationProviderId` = full type name of the provider, persisted per LDAP user. |
| Interfaces | `IAuthenticationProvider` (`Authenticate`, `ChangePassword`, `IsEnabled`), `IPasswordResetProvider`. |
| Provider order | `AuthenticateLocalUser` tries providers for an unknown user, then switches its `AuthenticationProviderId`. Implicit creation relies on this. |
| `IUserManager` API | `CreateUserAsync`, `UpdateUserAsync`, `GetUserById`, `GetUserByName`, `SetPermission` / `SetPreference`. |
| LDAP ↔ Jellyfin link | Stored in the plugin config (`LdapUsers[]`: LDAP uid → `LinkedJellyfinUserId`), not in the database. |
| Rights mapping | Admin via `LdapAdminFilter`, resynced on every login. Libraries applied on creation only. |
| Scheduled task | `LdapProfileImageSyncTask` (avatars). |

### Clients

- **Web**: the username/password form is hardcoded (`src/apps/legacy/controllers/session/login/index.js`). QuickConnect is shown only if `/QuickConnect/Enabled` is true. Both `jellyfin-apiclient` and `@jellyfin/sdk` are in use.
- **Android**: a WebView wrapper around jellyfin-web. Most IdPs, Google included, block OAuth in embedded WebViews ([RFC 8252](https://www.rfc-editor.org/rfc/rfc8252)), so the redirect flow must open in a Custom Tab and come back through a deep link.
- **Android TV**: native login, `AuthenticateMethod` = `Automatic | Credential | QuickConnect`. Ready for a new device-code method.
- **Others**: the TypeScript and Kotlin SDKs (generated from OpenAPI), Swiftfin, third-party clients, and the community [SSO plugin](https://github.com/9p4/jellyfin-plugin-sso).

### Gaps

1. No discovery: a client can't know which login methods exist.
2. The provider contract is username/password only: no redirect or device flow, no claims or groups.
3. No stable external identity (issuer + subject).
4. No shared mapping from claims to permissions and libraries: each plugin writes its own.
5. No device flow usable without an existing session (TV, consoles, third-party clients).
6. Auth is mixed into `UserManager`, which makes any change risky.
7. (Parallel, not blocking) Tokens have no expiry or fine-grained revocation.

## 3. Design decisions

| # | Decision | Consequences |
| --- | --- | --- |
| D1 | **Password login keeps the current mechanism, one form.** Local, LDAP and any other password provider appear as a single `password` entry in discovery. The server resolves the provider as today. | Clients never know about LDAP. `AuthenticateByName` and `ForgotPassword` are unchanged. |
| D2 | **LDAP before OIDC.** L.1 (LDAP feeds the claims pipeline) is merged before the OIDC plugin (3.4). | The pipeline is validated with real existing users first. If L.1 shows a flaw, 1.2 is fixed before going further. |
| D3 | **ASP.NET Core first, nothing custom where the framework already does it.** | An external provider is an `AuthenticationScheme` registered by a plugin. A login result is a `ClaimsPrincipal`. Each custom piece is justified (section 5). |
| D4 | **Claims → permissions mapping lives in the core.** | One contract for all plugins, present and future: a plugin only produces claims. Admins configure rights in one place. |
| D5 | **QuickConnect is merged into the device flow.** The device flow becomes the common base for any "log in from another device" (code, QR, ...). | QuickConnect stays as a compatibility layer. Clients implement one mechanism. |
| D6 | **New endpoints go under `/Users/*`, not `/Auth/*`.** | Consistent with `/Users/AuthenticateByName` and `/Users/AuthenticateWithQuickConnect`, and avoids clashing with the existing admin `GET /Auth/Providers`. Names below are proposals. |
| D7 | **ASP.NET Core Identity adopted piece by piece** with `AddIdentityCore<User>()` and stores on Jellyfin's tables. Never `AddIdentity()` or `MapIdentityApi`. | See section 6. |

Guiding principles:

- The core provides the plumbing, plugins provide the protocols. No OIDC-specific code in the core.
- Every flow ends with a normal Jellyfin token. `CustomAuthentication` is unchanged, existing clients keep working.
- LDAP never breaks: the current plugin must run without a rebuild on every intermediate version.
- Clients implement each flow type once (`password`, `redirect`, `device`), never each plugin.
- One proposal, then small PRs per step, each mergeable and useful on its own.

## 4. Need → ASP.NET Core piece

| Need | Framework piece | Remaining custom code |
| --- | --- | --- |
| Plugin adds an external provider | `IPluginServiceRegistrator` → `AddOpenIdConnect` / `AddOAuth` / third-party SAML handler. One `AuthenticationScheme` per provider, with its `DisplayName`. | None. |
| Discovery | `IAuthenticationSchemeProvider.GetAllSchemesAsync()`, filtered on remote handlers. | A thin `GET /Users/AuthenticationMethods` (DTO projection plus the aggregated `password` entry). |
| Start login | `ChallengeAsync(scheme, AuthenticationProperties { RedirectUri })`. The callback is handled by the handler (`CallbackPath`). | None. |
| Read the identity after callback | `SignInScheme` = Identity external cookie (`IdentityConstants.ExternalScheme`), read through `SignInManager.GetExternalLoginInfoAsync()`. | None. |
| One-time exchange code | Data Protection (`ITimeLimitedDataProtector`). | Single use: a small server-side cache of consumed codes. |
| Jellyfin token | Existing `SessionManager` and `CustomAuthentication`. | None. |
| External identity (issuer, subject) | Identity `IUserLoginStore<User>` (`AddLoginAsync`, `FindByLoginAsync`, `RemoveLoginAsync`) via `AddIdentityCore<User>()`. | The EF table (modeled on `IdentityUserLogin`) and its store. Custom stores are a supported Identity extension point. |
| Claims → rights | `IClaimsTransformation`, `ClaimActions` (plugin side), existing authorization policies. | The `group → admin / libraries` rules, which are Jellyfin concepts. |
| Password providers (local, LDAP) | No equivalent without adopting `SignInManager` for passwords. | Keep `IAuthenticationProvider` (D1). Optional addition: return a `ClaimsIdentity`. |
| Device flow (RFC 8628) | ASP.NET Core has no authorization server. | Extended QuickConnect (section 5). |
| Token expiry and refresh (not on the critical path) | `AddBearerToken()`, Data Protection. Not `MapIdentityApi`. | Separate proposal. |
| Add a provider without restart | `IAuthenticationSchemeProvider.AddScheme` / `RemoveScheme`, `IOptionsMonitorCache`. | A restart is acceptable in v1 (as in #17271). |

## 5. Where custom code remains, and why

### Password login (local, LDAP) keeps the current mechanism

ASP.NET Core authentication schemes either validate an incoming request (cookie, token) or redirect to an external provider. None of them takes a username and password. The only standard piece that does is ASP.NET Core Identity (`SignInManager.PasswordSignInAsync`), which brings its own user model and has no concept of pluggable password backends.

Jellyfin already has its own user entity, password hash, lockout, and the `IAuthenticationProvider` chain that LDAP plugs into. Moving LDAP into Identity would mean overriding `UserManager.CheckPasswordAsync` to call the existing providers, which is rebuilding the current chain inside Identity for no gain.

So:

- Password login doesn't change. No migration of users or hashes, no change for clients or the LDAP plugin.
- The only addition (L.1): a provider can also return a `ClaimsIdentity` with its groups. It then goes through the same identity-link and mapping pipeline as OIDC.
- This is consistent with D3: no new custom code is written. We keep what exists because the standard alternative costs much more.
- Known limit: password logins don't get Identity features (TOTP, passkeys) for free. That is a separate proposal.

### Device flow: extended QuickConnect rather than OpenIddict

This is the one place where the design suggests custom code while a library exists, so it needs a justification.

1. **D3 is not contradicted.** D3 is about what the framework already does. ASP.NET Core has no authorization server, and Microsoft's docs point to third-party products (Duende IdentityServer, OpenIddict). The choice is between a large third-party dependency and extending existing code.
2. **OpenIddict is oversized for this.** Even for the device flow alone, it needs its stores and EF tables (applications, authorizations, scopes, tokens), each client registered as an OAuth application, its own endpoints, and above all its own token format. That means two token systems side by side, or a bridge between them, which goes against "every flow ends with a normal Jellyfin token". Adopting it effectively makes Jellyfin an identity provider, a much bigger decision that would deserve its own proposal.
3. **QuickConnect already has the shape of a device flow:**

   | RFC 8628 | QuickConnect today |
   | --- | --- |
   | Authorization request → `device_code` + `user_code` | `POST /QuickConnect/Initiate` → `Secret` + `Code` |
   | Client polling | `GET /QuickConnect/Connect?secret=...` |
   | User approval on another device | `POST /QuickConnect/Authorize` |
   | Token exchange | `POST /Users/AuthenticateWithQuickConnect` → Jellyfin `AuthenticationResult` |
   | `expires_in` | 10 min TTL |

   Missing pieces: `verification_uri` / `verification_uri_complete` (for QR codes), `interval` and `slow_down`, rate limiting against brute force of the 6-digit code, and approval after login with any provider (which comes with phase 3).
4. **The custom part stays small and uses standard pieces:** Data Protection for the `device_code`, `Microsoft.AspNetCore.RateLimiting` for polling and code entry, existing `SessionManager` for the token.
5. **The door stays open.** Field names and errors follow RFC 8628 (`authorization_pending`, `slow_down`, `expired_token`, `access_denied`). If Jellyfin ever becomes a full OIDC provider, moving to OpenIddict wouldn't change client behavior.

## 6. ASP.NET Core Identity: what we take, what we leave

Based on the ASP.NET Core 10 docs. Identity fits in part: it covers external logins, claims, 2FA and passkeys, but not LDAP, Jellyfin rights mapping, or the device flow. It is adopted piece by piece.

| Need | Identity (.NET 10) | Verdict |
| --- | --- | --- |
| External identity link | `IUserLoginStore` (`LoginProvider`, `ProviderKey`), `SignInManager.GetExternalLoginInfoAsync`, external cookie | Adopted (2.0, 2.2, 3.2, 3.3). |
| Keep the Jellyfin database | Custom stores: only `IUserStore<TUser>` is required. Identity's EF tables aren't needed. | Adopted, on the existing `User` entity, no user migration. |
| Identity as claims | `IUserClaimsPrincipalFactory`, `IUserClaimStore`, `IUserRoleStore` | Usable for the pipeline and phase 5. |
| IdP groups → admin and libraries | Nothing: roles don't know about Jellyfin libraries. | Stays Jellyfin code. |
| Local password | `CheckPasswordAsync`, `IPasswordHasher` | Possible with an adapter. Not done (D1). |
| LDAP / several password backends | No provider chain. | Confirms D1. |
| Lockout | `IUserLockoutStore`, temporary lockout | Jellyfin disables the account instead. Keep the current behavior for now. |
| TOTP and recovery codes | `IUserAuthenticatorKeyStore`, `IUserTwoFactorRecoveryCodeStore` | Later (2FA proposal). |
| Passkeys (new in .NET 10) | `SignInManager.PasskeySignInAsync`, `IUserPasskeyStore` | Later, browser side. No attestation validation by default, little use on TV apps. |
| Expiring tokens with refresh | `MapIdentityApi` | Not usable: email login, fixed endpoints, no per-device sessions. `AddBearerToken()` alone may be considered later. |
| Device flow, OIDC server | Nothing (docs point to Duende IdentityServer). | See section 5. |

Pitfalls, used as review criteria:

- Don't call `AddIdentity()`: it replaces the default scheme with its cookies and would break `CustomAuthentication` and every client. Use `AddIdentityCore<User>()` and add only the external cookie.
- Don't use `MapIdentityApi`.

Sources:

- [Passkeys in ASP.NET Core Identity](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/passkeys/?view=aspnetcore-10.0)
- [Custom storage providers for ASP.NET Core Identity](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity-custom-storage-providers?view=aspnetcore-10.0)
- [Identity API endpoints](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity-api-authorization?view=aspnetcore-10.0)

## 7. Plan

Repos: server, plugin, SDK, web, Android / Android TV. Size: S < 300 lines, M < 800 lines, L > 800 (to avoid).

### Phase 0: agreement (no code)

| # | Deliverable | Repo |
| --- | --- | --- |
| P0.1 | Proposal issue with this document. | jellyfin-meta |
| P0.2 | Agree on D4 to D7, the suggestions in section 9, and the external login table shape. | jellyfin-meta |

### Phase 1: backend cleanup, no behavior change

| # | PR | Repo | Size | Depends on |
| --- | --- | --- | --- | --- |
| 1.0 | LDAP safety net: integration tests with a fake legacy provider reproducing LDAP behavior (unknown user → created by the provider, `AuthenticationProviderId` switch, `ChangePassword`, reset through `IPasswordResetProvider`, lockout). | server | M | P0 |
| 1.1 | Move auth out of `UserManager` into a dedicated service. Pure refactor. | server | M | 1.0 |
| 1.2 | Shared `ClaimsPrincipal` → Jellyfin user pipeline: one entry point taking a principal from an external scheme or a password provider, resolving or creating the user. Password providers may optionally return a `ClaimsIdentity`. No new provider interface. | server | M | 1.1 |
| 1.3 | Legacy compatibility: `IAuthenticationProvider` goes through the pipeline with the same provider IDs, order and fallback, and no DB rewrite. | server | S | 1.2 |
| 1.4 | CI job: build the published LDAP plugin unchanged against preview packages, test it against an OpenLDAP container (login, auto-creation, admin filter, password change). | plugin | M | 1.0 |

### Phase 2: discovery and external identities

| # | PR | Repo | Size | Depends on |
| --- | --- | --- | --- | --- |
| 2.0 | Minimal Identity base: `AddIdentityCore<User>()`, `IUserStore<User>` on the existing entity, external cookie only as `SignInScheme` for plugins. Documented convention for plugins. No change to password login. | server | M | 1.2 |
| 2.1 | `GET /Users/AuthenticationMethods` (anonymous): external schemes as `{id, displayName, flowType}` plus a single `password` entry. The admin `GET /Auth/Providers` is untouched. | server | S | 2.0 |
| 2.2 | External login table modeled on `IdentityUserLogin` (`UserId`, `LoginProvider`, `ProviderKey`, `ProviderDisplayName`, `LastLogin`), EF migration, `IUserLoginStore<User>`. | server | M | 2.0 |
| 2.3 | `/Users/{userId}/ExternalLogins`: list and unlink. | server | S | 2.2 |
| 2.4 | Regenerate the TS and Kotlin SDKs. | SDK | S | 2.1, 2.3 |
| 2.5 | Web login page driven by 2.1: one username/password form if a `password` entry exists, one button per external method. | web | M | 2.4 |

### Phase L: LDAP on the new pipeline (`jellyfin-plugin-ldapauth`)

Every step stays optional for admins: without action on their side, behavior is identical.

| # | PR | Size | Depends on |
| --- | --- | --- | --- |
| L.1 | Keep `IAuthenticationProvider` and also return a `ClaimsIdentity`: `NameIdentifier` = `LdapUidAttribute` value, issuer = server URI and base DN, group claims = LDAP groups. Same form, same resolution, same implicit creation. Merged before 3.4 (D2). | M | 1.2, 1.3 |
| L.2 | Copy `LdapUsers[]` to the external login table (2.2) at startup. Idempotent; the old config is kept read-only for rollback. | M | 2.2, L.1 |
| L.3 | Convert `LdapAdminFilter` into a core rule and folder settings into the provider's default rule, behind a "use Jellyfin mapping" switch, off by default. | M | 5.1, 5.2, L.1 |
| L.4 | Password reset (`PasswordResetUrl`) and avatar sync stay in the plugin. | S | L.1 |
| L.5 | Mark `IHasNewUserPolicy` `[Obsolete]` (replaced by 5.1 and 5.3) once L.1 to L.3 are released and one major version has passed. `IAuthenticationProvider`, `IPasswordResetProvider` and `IRequiresResolvedUser` stay (D1). | S | L.3 |

### Phase 3: redirect flow (web and mobile)

| # | PR | Repo | Size | Depends on |
| --- | --- | --- | --- | --- |
| 3.1 | `GET /Users/ExternalLogin/{scheme}/Start`: a plain `ChallengeAsync(scheme)`. The plugin's handler processes the callback and writes the external cookie. | server | S | 2.1 |
| 3.2 | `GET /Users/ExternalLogin/Complete`: `GetExternalLoginInfoAsync()`, `FindByLoginAsync()`, pipeline 1.2, one-time code, redirect to the client. `POST /Users/AuthenticateWithExternalCode` returns a normal `AuthenticationResult`; session and device are created only then. Taken from #17271. | server | M | 3.1, 2.2 |
| 3.3 | Account linking: a logged-in user links an external login (`ConfigureExternalAuthenticationProperties` with the user id, then `AddLoginAsync`). | server | S | 3.2 |
| 3.4 | OIDC plugin: registers one `AddOpenIdConnect` scheme per configured IdP, with `ClaimActions` for groups, plus its config page. The ASP.NET Core part of #17271 fits here. | plugin | M/L | 3.2, L.1 |
| 3.5 | Web callback page: exchange the code, store the token. | web | S | 3.2, 2.4 |
| 3.6 | Android: open `Start` in a Custom Tab, come back to the WebView through a deep link. | Android | M | 3.5 |

### Phase 4: device flow (TV, consoles, third-party clients)

| # | PR | Repo | Size | Depends on |
| --- | --- | --- | --- | --- |
| 4.1 | Device authorization on extended QuickConnect (section 5): RFC 8628 fields and errors, `verification_uri_complete` for QR, Data Protection, rate limiting. Approval with any provider. | server | M | 3.2, P0.2 |
| 4.2 | QuickConnect merged into the device flow (D5): `/QuickConnect/*` and `AuthenticateWithQuickConnect` kept as a compatibility layer. No change for current clients. | server | M | 4.1 |
| 4.3 | Web approval page for `verification_uri`. | web | S | 4.1 |
| 4.4 | Android TV: new device-code method with QR and code, based on the QuickConnect screen. | Android TV | M | 4.1 |

### Phase 5: claims → Jellyfin permissions

| # | PR | Repo | Size | Depends on |
| --- | --- | --- | --- | --- |
| 5.1 | Rule model in the core (D4): `{provider, claim/group} → {admin, library access, permissions, required to log in}`, applied to the `ClaimsPrincipal`. Reuses the `Group` entity (to confirm). | server | M | 1.2 |
| 5.2 | Applied on every login, with last-admin protection and ActivityLog entries. | server | M | 5.1, 3.2 |
| 5.3 | Provisioning policy per provider: auto-create, link only, deny. | server | S | 5.2 |
| 5.4 | Admin UI: providers list and mapping rule editor. Protocol config stays in each plugin's page. | web | M | 5.1 |

### Acceptance criteria for every server PR

- The 1.0 tests pass.
- The 1.4 CI job is green against the latest released LDAP plugin, not only master.
- No `AuthenticationProviderId` is rewritten without an explicit, reversible migration.
- The PR says which ASP.NET Core piece it uses, or why none fits (D3).
- No `AddIdentity()` or `MapIdentityApi()`.

### Out of scope (separate proposals)

- Token expiry, rotation, revocation and hashing (`Device.AccessToken`): evaluate `AddBearerToken()`.
- 2FA and passkeys for local users on the Identity base from 2.0, instead of custom TOTP as in #14729. Clients will need to send the 2FA code.
- Proxy header auth ([jellyfin-meta#103](https://github.com/jellyfin/jellyfin-meta/issues/103)): one more scheme through the same pipeline.
- Documentation of flow types for Swiftfin and third-party clients.
- User documentation for common IdPs (Authelia, Authentik, Keycloak, ...).

## 8. Dependency graph

```text
P0 ─► 1.0 (LDAP tests) ─► 1.4 (LDAP plugin CI)
       │
       └─► 1.1 ─► 1.2 (pipeline) ─► 1.3 ─┬─► 2.0 (Identity base) ─► 2.1 ─► 3.1 ─► 3.2 ─┬─► 3.3
                                         │        └─► 2.2 (IUserLoginStore) ─► (needed by 3.2)
                                         │                                           ├─► 3.4 (OIDC plugin)
                                         │                                           ├─► 3.5 ─► 3.6
                                         │                                           └─► 4.1 ─► 4.2 / 4.3 / 4.4
                                         ├─► L.1 ─► L.2 (after 2.2) ─► L.4
                                         │    └──── blocks 3.4 (D2)
                                         └─► 5.1 ─► 5.2 ─► 5.3 / 5.4 ─► L.3 ─► L.5
```

First user-facing milestone: 1.0 → 1.3 → L.1 → 2.0 → 2.2 → 2.1 → 3.1 → 3.2 + 3.4 + 3.5, i.e. OIDC login on the web through a plugin. Second milestone: phase 4, login on TV and native clients by code or QR.

## 9. Questions

### Suggestions to confirm

| Question | Suggestion | To confirm with |
| --- | --- | --- |
| Reuse or remove the `Group` entity? | Reuse it for mapping rules (5.1). | Server maintainers |
| Target version? | 13.0 for phases 1 and 2. | Release team |
| LDAP CI job (1.4) and moving `LdapUsers[]` to the core (L.2)? | Yes to both. | LDAP plugin maintainers |
| Device flow engine? | Extended QuickConnect (section 5). | Server maintainers |

### Open

- Would an official OIDC plugin under the jellyfin org be acceptable? Who would maintain it?
- How many major versions between marking `IHasNewUserPolicy` `[Obsolete]` and removing it?

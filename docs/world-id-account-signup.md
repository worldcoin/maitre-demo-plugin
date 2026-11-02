# Google or World ID account signup

Status: implementation handoff for a draft PR. This repository contains plugin
manifests and agent instructions, not the Maître backend. No MCP handler, database
migration, or Auth configuration is changed by this PR. Backend integration and
end-to-end verification are required before changing the booking skill.

## Outcome and scope

Maître's normal MCP OAuth connection offers **Continue with Google** and
**Continue with World ID** for both signup and returning users. Either provider
creates or resolves the Supabase Auth user. Neither requires signing in with the
other provider to create an account or authorize the MCP connection. XAA, token
delegation from the World ID plugin, and a custom device-flow credential system
are outside this implementation.

Supabase provisions the Auth user during its provider callback; a separate
`create_account` tool is not required. Reuse existing profile provisioning. Only
add the optional idempotent tool below if application onboarding needs it.

Google login remains a permanent supported option. Keep the World verification
flow for Google users when a booking requires it. Signing up through either
provider does not authorize a reservation.

## User flow

1. The agent starts the Maître MCP connection through the host's OAuth controls.
2. Maître's authorization page offers Google and World ID side by side.
3. The person chooses and completes one provider's sign-in. For World-first
   signup, require at least one QR scan and World proof approval. Verify the
   deployed IDP's supported proof controls; fresh login alone can allow passkeys
   and does not guarantee a QR scan. Do not claim this requirement works until
   an end-to-end test confirms it.
4. Supabase's provider callback creates or resolves the Auth user. Maître
   provisions the application profile and resumes the pending MCP authorization.
5. The person approves agent access; the host completes its code exchange and
   stores the Maître credential. The agent continues with authenticated tools.
6. A Google-only user can use their account without linking World. If they select
   a booking requiring World verification, run the existing World linking flow
   against that same account before accepting the booking. World sign-in must
   also satisfy the backend's assurance policy before granting verified access.

## Observed deployment

Read-only public discovery checks on November 2, 2026 returned:

| Setting | Observed value |
| --- | --- |
| Maître MCP resource | `https://www.maitre.fun/mcp` |
| Supabase project ref | `axvzindzhqiuinvwxxgw` |
| Authorization server | `https://axvzindzhqiuinvwxxgw.supabase.co/auth/v1` |
| MCP scopes advertised by Maître | `openid` |
| Supabase authorization endpoint | `https://axvzindzhqiuinvwxxgw.supabase.co/auth/v1/oauth/authorize` |
| Supabase token endpoint | `https://axvzindzhqiuinvwxxgw.supabase.co/auth/v1/oauth/token` |
| Supabase client registration endpoint | `https://axvzindzhqiuinvwxxgw.supabase.co/auth/v1/oauth/clients/register` |
| World issuer | `https://auth.world.org` |
| World scopes advertised | `openid` |
| World subject type / signing algorithm / PKCE | pairwise / RS256 / S256 |

Supabase project access subsequently became available. Read-only inspection
confirmed the active project is `maitre`, with 16 `custom:world` identities and
14 `google` identities. These are identity counts, not distinct-user counts.
The project has no deployed Supabase Edge Functions, so this connection does not
provide the external Maître backend source. Reuse the existing `custom:world`
provider; do not introduce `custom:world-id`. Provider settings, application
schema and the complete MCP authorization path remain unverified. Public metadata
does not establish that client registration succeeds.

## Backend implementation

1. Inspect the current MCP handler, Supabase authorization UI and callback,
   application profile table, Auth-user triggers, verification mapping, and RLS.
   Reuse their conventions rather than introducing a second MCP server or a new
   profile table before inspecting the existing schema.
2. Offer Google and World ID sign-in in the existing authorization UI. Use the
   existing `google` provider and inspect/reuse the deployed World provider
   identifier `custom:world`, confirmed in `auth.identities`.
   Save the pending downstream OAuth authorization ID across
   upstream sign-in. Use a fixed, allowlisted application callback, exchange the
   Supabase login code there, and resume the original authorization request and
   consent. Never accept an arbitrary return URL from a tool argument.
3. Keep both authorization-code exchanges separate: the selected upstream
   provider to Supabase, and Supabase to the MCP host. Each has its own code, callback, state and PKCE
   verifier. The host stores Maître credentials outside model context.
4. Require a valid Maître MCP access token before running `create_account`.
   Unauthenticated requests use the existing HTTP OAuth challenge and protected
   resource metadata. Verify issuer, audience, expiry and the applicable
   client/resource permissions using the current authentication middleware.
5. Resolve the user UUID from that verified authentication context. Accept either
   Google or World authentication for account/profile setup. Do not require a
   linked World identity for Google signup or MCP connection. Gate only operations
   that require World verification through the existing trusted verification flow.
   Do not authorize using user-editable `user_metadata` or caller-supplied IDs.
6. Insert the application profile once, keyed by Supabase user UUID. Use the
   existing schema and a unique constraint with `ON CONFLICT DO NOTHING`, followed
   by a read. A retry must not overwrite an existing name or create another user.
   Use a transaction/RPC if multiple application rows must be created together.
7. Return `ready` only after the profile read succeeds. An uncertain database
   result is retried using the same authenticated UUID; it must not cause another
   Auth-user creation. Never call `admin.createUser()` or insert into `auth.users`
   for this flow: Supabase's provider login owns Auth-user provisioning.
8. Preserve explicit identity linking for existing Google users. Link World while
   authenticated to the existing account; retain reservations and UUID. An identity
   already linked elsewhere is a conflict to resolve, not permission to merge.

Optional profile-onboarding tool contract (only if needed by the existing backend):

```ts
type CreateAccountInput = {
  display_name?: string; // Trim; require 1–100 characters if present.
};

type CreateAccountResult = {
  status: "ready";
  profile_created: boolean;
};
```

Do not accept `user_id`, World `sub`, issuer, token, verification status, or
callback URL as tool inputs. Keep World IDs and all credentials out of results.
Mark the tool as a non-destructive, idempotent write. It must not book a table.

The Auth user exists before this handler runs. `profile_created` describes only
the application profile, not whether Supabase just created its Auth user.

World identity is the validated `(iss, sub)` pair, with opaque, case-sensitive
`sub`. Supabase keeps its UUID as the application key. Keep the existing World
client/sector where possible: a new sector changes pairwise subjects and requires
explicit account linking. Name and email are not required for account creation.
Ask for a guest name at booking if the profile has none.

Account recognition does not by itself satisfy a booking assurance policy. Keep
verification decisions in trusted backend state derived from validated World
claims. Confirm how Supabase preserves `acr`, `amr` and `auth_time` before replacing
the current verification check. A provider name or a refreshed Supabase session
does not establish fresh World authentication.

## Manual setup checklist

These steps are pending inspection, not confirmed missing configuration. Preserve
working provider settings and Google access while reviewing them.

### World developer portal

- Identify Maître's existing production client and identity sector. Prefer
  reusing them to preserve subjects. Do not use the World plugin's client.
- Register the exact callback URL displayed by Supabase's custom-provider form.
  It is normally `https://axvzindzhqiuinvwxxgw.supabase.co/auth/v1/callback`;
  use the form's actual value, including any provider-specific query parameters.
- Match the client's registered token-endpoint authentication method with the
  Supabase connector. Store the client secret only in Supabase's provider config.
  Rotate credentials without changing sectors. Never paste secrets into chat.

### Supabase Dashboard

Open the [Maître project](https://supabase.com/dashboard/project/axvzindzhqiuinvwxxgw).

1. In **Authentication → Providers**, inspect the existing `custom:world`
   provider and reuse it. Keep the existing client and identity sector. Compare
   its settings against the target below; do not create a second World provider:

   ```json
   {
     "provider_type": "oidc",
     "identifier": "custom:world",
     "name": "World ID",
     "issuer": "https://auth.world.org",
     "scopes": ["openid"],
     "email_optional": true,
     "pkce_enabled": true,
     "skip_nonce_check": false
   }
   ```

   Enter the production client ID and secret securely. If advanced fields are
   unavailable in the UI, set them through the documented server-side custom
   provider Admin API with a compatible pinned SDK. Enable the provider only after
   the World callback and client authentication method match. Verify actual
   upstream requests contain S256 PKCE and no added `profile` or `offline_access`.
   Keep the existing Google provider enabled with working credentials and its
   registered Supabase callback. Both provider buttons must resume the same
   pending MCP authorization. No provider migration is needed merely to add the
   second option to the authorization UI.
2. In **Authentication → URL Configuration**, allowlist the exact application
   callback implemented by the backend. This callback is distinct from the
   Supabase callback registered at World and the host's downstream OAuth callback.
   Confirm Site URL and production redirects; avoid broad wildcards.
3. In **Authentication → OAuth Server**, inspect the existing configuration:
   public metadata already advertises this server. Preserve its authorization UI
   path, client registrations and consent behavior. Confirm each supported host
   can register or has a pre-registered client. Do not enable unrestricted dynamic
   registration merely because it is advertised.
4. Enable **manual identity linking** if not already enabled, so signed-in Google
   users can attach their World identity without losing their application UUID.
   Test `linkIdentity` with the selected provider before offering migration.
5. Inspect application tables, grants and RLS before applying the backend's
   migration. Profiles need one unique row per Auth UUID; users may read only their
   own profile. If direct authenticated inserts are allowed, enforce ownership in
   `WITH CHECK`. Restrict updates to editable profile fields, never verification
   or ownership fields. Check existing Auth-user triggers tolerate absent name
   and email. Run Supabase security advisors after schema changes.
6. Keep sandbox issuer, client registration and account mappings isolated in a
   development project. Do not replace production World settings with sandbox
   credentials to test.

## Verification and rollout

- Fresh World-only signup, including no email: one `auth.users` row, one World
  provider identity and one application profile. Confirm a read resolves to the
  same Supabase UUID through the authorized MCP connection. Confirm the first
  World signup requires a QR scan, including when an IDP session or passkey exists.
- Fresh Google-only signup: one Auth user and profile, with a working MCP
  connection and no World-linking requirement. World-gated bookings remain blocked
  until explicit verification succeeds against that account.
- Returning Google and World users: both provider buttons resolve their existing
  account and resume agent authorization without creating duplicates.
- Repeat login and concurrent `create_account` calls: the same UUID and profile,
  with the existing display name preserved.
- Existing Google account: explicit World linking preserves UUID and reservations;
  a conflicting World identity fails without merging accounts.
- Reject expired/invalid tokens, wrong issuer/audience/resource, foreign profiles,
  forged metadata, callback/state/PKCE failures and callback replay. Cancelled
  sign-in grants no MCP access or application profile setup. Auth-user creation
  during upstream sign-in may precede downstream consent; it grants no MCP access.
- Confirm World signup does not bypass the booking assurance policy; names are
  requested only when needed. Reconnect, refresh and logout follow existing MCP
  behavior. World logout does not automatically revoke Maître sessions.
- After staging passes, deploy the backend and provider settings together. Update
  this plugin's booking skill and README to describe both signup options only
  after the live flows work. Keep Google supported and retain World-verification
  recovery for Google accounts and rollback.

No live signup, booking, provider mutation or database write was performed while
preparing this handoff. Public discovery checks are not an end-to-end Auth test.

## References

- [Supabase custom OAuth/OIDC providers](https://supabase.com/docs/guides/auth/custom-oauth-providers)
- [Supabase MCP authentication](https://supabase.com/docs/guides/auth/oauth-server/mcp-authentication)
- [Supabase identity linking](https://supabase.com/docs/guides/auth/auth-identity-linking)
- [World auth-server federation](https://github.com/worldcoin/temporal-apps/blob/main/apps/idpapi/mcp/guides/federation.md)
- [World identity contract](https://github.com/worldcoin/temporal-apps/blob/main/apps/idpapi/mcp/guides/id-token.md)
- [World application registration and sectors](https://github.com/worldcoin/temporal-apps/blob/main/apps/idpapi/mcp/guides/app-registration.md)

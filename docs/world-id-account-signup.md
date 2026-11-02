# World ID account signup

Status: implementation handoff for a draft PR. This repository contains plugin
manifests and agent instructions, not the Maître backend. No MCP handler, database
migration, or Auth configuration is changed by this PR. Backend integration and
end-to-end verification are required before changing the booking skill.

## Outcome and scope

An agent can request `create_account` and use Maître's normal MCP OAuth connection
to sign in with World ID. Supabase creates or resolves the Auth user; the tool
idempotently finishes application profile setup. Google is optional for new
accounts. XAA, token delegation from the World ID plugin, and a custom device-flow
credential system are outside this implementation.

Keep the existing Google login and World verification recovery available during
rollout. Signing up does not authorize a reservation.

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

Metadata establishes endpoint discovery, not that client registration succeeds or
that World is configured as a Supabase provider. The installed Supabase connector
did not expose project/query tools in the authoring chat; provider configuration,
schema, migrations, and deployed function source have not been inspected.

## Backend implementation

1. Inspect the current MCP handler, Supabase authorization UI and callback,
   application profile table, Auth-user triggers, verification mapping, and RLS.
   Reuse their conventions rather than introducing a second MCP server or a new
   profile table before inspecting the existing schema.
2. Add World ID sign-in to the existing authorization UI, using provider
   `custom:world-id`. Save the pending downstream OAuth authorization ID across
   upstream sign-in. Use a fixed, allowlisted application callback, exchange the
   Supabase login code there, and resume the original authorization request and
   consent. Never accept an arbitrary return URL from a tool argument.
3. Keep both authorization-code exchanges separate: World to Supabase, and
   Supabase to the MCP host. Each has its own code, callback, state and PKCE
   verifier. The host stores Maître credentials outside model context.
4. Require a valid Maître MCP access token before running `create_account`.
   Unauthenticated requests use the existing HTTP OAuth challenge and protected
   resource metadata. Verify issuer, audience, expiry and the applicable
   client/resource permissions using the current authentication middleware.
5. Resolve the user UUID from that verified authentication context. Require a
   trusted linked World identity for this signup path. A Google-only session must
   link World through the existing verification flow or complete World sign-in.
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

Proposed tool contract (to register in the existing backend):

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

1. In **Authentication → Providers**, inspect any existing World provider. Reuse
   its identifier in backend code if it already implements the intended OIDC
   contract; do not create a duplicate identity namespace casually. Otherwise add
   an **Auto-discovery (OIDC)** provider with identifier `custom:world-id`:

   ```json
   {
     "provider_type": "oidc",
     "identifier": "custom:world-id",
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
  same Supabase UUID through the authorized MCP connection.
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
  this plugin's booking skill and README to describe World-primary signup only
  after the live flow works. Retain the Google/verification fallback for rollback.

No live signup, booking, provider mutation or database write was performed while
preparing this handoff. Public discovery checks are not an end-to-end Auth test.

## References

- [Supabase custom OAuth/OIDC providers](https://supabase.com/docs/guides/auth/custom-oauth-providers)
- [Supabase MCP authentication](https://supabase.com/docs/guides/auth/oauth-server/mcp-authentication)
- [Supabase identity linking](https://supabase.com/docs/guides/auth/auth-identity-linking)
- [World auth-server federation](https://github.com/worldcoin/temporal-apps/blob/main/apps/idpapi/mcp/guides/federation.md)
- [World identity contract](https://github.com/worldcoin/temporal-apps/blob/main/apps/idpapi/mcp/guides/id-token.md)
- [World application registration and sectors](https://github.com/worldcoin/temporal-apps/blob/main/apps/idpapi/mcp/guides/app-registration.md)

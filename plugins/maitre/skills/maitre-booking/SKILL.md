---
name: maitre-booking
description: Set up a Maître account and find, claim, view, or cancel restaurant reservations. Use for Maître sign-in, tables, or claiming the Maître World ID benefit.
---

# Maître bookings

Maître sets aside restaurant tables for verified humans in San Francisco. Use
this plugin's connected Maître MCP tools at `https://www.maitre.fun/mcp` for live
availability and reservations.

## World ID benefits

For a request about the Maître World ID benefit, reuse the selected World ID
production catalog listing from the conversation. If it is missing, use the
production World ID plugin's `get_benefits` tool at `https://auth.world.org/mcp`.
Do not substitute the sandbox catalog or treat its listings as production offers.
If the production connection or catalog is unavailable, or Maître is not listed,
say so; direct Maître browsing remains available.

Use the listing's actual terms. Access to tables does not imply a discount or a
free meal. A listing or successful World ID account check does not establish
Maître eligibility, authorize a booking, or confirm redemption. Never pass World
ID tokens, proofs, or account identifiers to Maître as authorization.

If Maître tools are missing, use the host's tool/plugin discovery to find the
connection. Explain any required setup and resume with MCP when connected. Use
the website for authorization, an explicitly requested website flow, or when no
usable MCP route exists; explain the reason for a fallback. An uncertain write
must be resolved before trying another route.

## Browse and claim

1. Call `list_seats` for current availability. Its optional `day` is YYYY-MM-DD
   in America/Los_Angeles. Filter the returned options by the user's preferences;
   never invent availability or seat IDs.
2. Obtain the user's selection of restaurant, local date/time, and party size.
   A clear request to book those exact details already supplies approval. Omit
   `guest_name` to use their Google account name, or pass an override they request.
   Ask for a name only if the tool reports that one is missing or unavailable.
3. If this conversation has not established Maître access, check `my_reservations`
   and guide sign-in below before submitting the claim. Then call `claim_seat`
   with the selected `seat_id` and optional `guest_name`. Follow the World ID
   verification flow below when required.
4. Report the returned restaurant, local date/time, party size, actual guest name,
   claim reference, and status. `pending` means the claim was submitted to Maître
   and restaurant confirmation is outstanding. Do not call it a confirmed
   restaurant reservation.

For a seat conflict, refresh availability and let the user choose a replacement.
For a timeout or unknown write result, check `my_reservations` before retrying.
If the outcome remains ambiguous, report that uncertainty and stop writes;
do not automatically submit another claim or book a different table.

## Sign-in and World ID verification

Booking requires a Maître account created by signing in with Google, access for
this agent to that account, and World ID connected to that same Maître account.
Connecting the World ID plugin alone does not complete these steps. Browsing
tables is public; do not require account setup just to browse.

When Maître sign-in is needed, explain: "First, sign in to Maître with Google.
This creates your account if you're new. Then we'll connect World ID to that
account and continue with your table." Skip steps already confirmed complete.
Describe this as connecting their Maître account, without MCP server terminology.
Never ask for passwords or tokens in chat.

- Start the host's available Maître Connect/Reconnect flow. Its browser flow
  includes creating or signing in to Maître with Google, then allowing the agent access.
  Show the actual authorization URL as **Connect Maître** when one is returned.
- If native connection controls are unavailable but this Codex environment has
  a local shell and supports `codex mcp login`, run `codex mcp login maitre` on
  the user's behalf (use the actual configured server name if different).
  Show its returned authorization link and wait for browser completion. Do not
  merely hand the user a terminal command when you can start login yourself.
- If you cannot start either connection flow, give a concrete first step:
  [Create or sign in to your Maître account](https://www.maitre.fun/auth.html).
  Tell them to choose **Sign in with Google**, then return to chat. Explain that
  the plugin's Connect/Reconnect step is still needed to authorize this agent;
  website sign-in alone does not connect it. Do not invent an OAuth URL or say a
  popup opened without evidence.

After login, use `my_reservations` to check access, with at most one additional
read-only check after a supported reconnect. Preserve the selected table and
existing booking approval while the user signs in. Resume in the current chat
when supported; suggest a new chat only if the host cannot reload the connection.
Do not use booking or cancellation to test login, and do not claim the agent is
connected merely because the website is signed in.

`claim_seat` may return `verification_required` with `verification_url` and
`handoff`, even when the tool marks the result as an error. This means MCP
sign-in succeeded but this Maître account still needs World ID verification:

- Show the exact returned URL as a clickable **Connect World ID** link. Tell
  the user to use the same Google account as their Maître MCP connection.
- Call `wait_for_verification` with the returned `handoff`. On `pending`, call
  it again with the same handoff while the agent turn remains active.
- On `verification_required`, show the replacement URL and use its new handoff.
- On `verified`, retry the already-approved claim without requiring another
  user message. Verification does not hold the table; availability may change.
- On `timed_out`, `verification_recovery_required`, or another error, stop
  polling and show any returned recovery URL. Do not create a fresh handoff
  merely to extend the five-minute waiting window. Resume after the user asks.

Do not restart MCP OAuth for World ID verification. Verification is enforced by
Maître's backend; the skill cannot transfer the World ID plugin's credentials
or bypass this check. Waiting does not wake a stopped conversation.

## Manage reservations

Use `my_reservations` for the signed-in user's claim IDs and current statuses.
Cancel only a specific reservation the user has asked to cancel, using its
`claim_id` with `cancel_reservation`.

- `cancelled`: report cancellation complete.
- `cancellation_requested`: explain that restaurant cancellation is still
  outstanding and the reservation remains active.
- Unknown result: check `my_reservations`; do not report success or book a
  replacement until cancellation is confirmed.

Use `logout` only when the user asks to sign out or disconnect this Maître MCP
connection. It does not cancel reservations or disconnect their World ID.
After successful logout, stop authenticated calls until they ask to reconnect.

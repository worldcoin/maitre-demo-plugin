# Maître plugin

Find and claim restaurant tables for verified humans, alongside the
[World ID plugin](https://github.com/worldcoin/world-id-agent-plugin).

This small plugin connects to Maître's existing MCP server at
`https://www.maitre.fun/mcp` and bundles one booking skill. No local server,
API keys, or build step is required.

## Install in ChatGPT desktop

Open this repository in the ChatGPT desktop app, then select **Maître Demo**
as the source in the Plugins Directory and install **Maître**. If the source
does not appear, restart the desktop app after opening the repository.
Start a new conversation after installation to load the tools and skill.

Once these files are pushed to GitHub, give a coding agent this prompt:

```text
Install the Maître plugin from https://github.com/worldcoin/maitre-demo-plugin
```

The repository marketplace is `.agents/plugins/marketplace.json`; its plugin
is `maitre`, under `plugins/maitre`.

For Codex CLI, install from a local checkout:

```sh
codex plugin marketplace add /absolute/path/to/maitre-demo-plugin
codex plugin add maitre@maitre-demo
codex mcp login maitre
```

Or, after pushing the plugin files, use
`codex plugin marketplace add worldcoin/maitre-demo-plugin` for the first command.
Complete the browser authorization and start a new session.

This repo bundles its MCP connection directly, like the World ID plugin; no
registered ChatGPT app ID is needed for the desktop route. OpenAI marks
repo-imported plugins with bundled MCP servers **Desktop only**, including remote
HTTPS servers. ChatGPT web uses a separate registered-app setup; see
[OpenAI's import guidance](https://learn.chatgpt.com/docs/enterprise/plugin-management#desktop-only-plugins)
and [plugin packaging](https://developers.openai.com/plugins/build/plugins).

## Use with World ID

Install the production World ID plugin (`world-id`) from its `main` branch
separately. It uses `https://auth.world.org/mcp`. Installing this plugin does
not install World ID or change its environment.

For discovery through World ID, Maître needs an approved, published listing in
the production benefit catalog. An empty catalog does not prevent direct Maître
booking, but the agent must not claim a listed offer exists or substitute the
sandbox catalog. Maître's Supabase World ID provider must also use the production
issuer, `https://auth.world.org`.

Try:

```text
What World ID dining benefits are available?
Help me claim the Maître benefit. Find a table for two this Friday.
Show my Maître reservations.
```

For direct Maître availability, the agent calls `list_seats`. For a benefit
request, it uses the World ID listing, then Maître's tools to select and claim
a table. The benefit is access to tables for verified humans; a free meal or
discount is not implied.

Maître has its own Google/Supabase OAuth connection. If the account is not yet
verified, the agent displays a **Connect World ID** link and waits while you
verify using the same Google account. It then retries the selected booking.
Connecting the World ID plugin alone does not authorize Maître. Cross-app
credential delegation is not implemented by the current backend.

A successful claim currently returns **pending restaurant confirmation**.
The agent must distinguish this from a confirmed restaurant reservation.
Cancellation can likewise remain pending until the restaurant confirms it.

## Package

```text
.agents/plugins/marketplace.json
plugins/maitre/
  plugin.json                  Portable plugin identity
  mcp.json                     Portable remote MCP configuration
  .codex-plugin/plugin.json    OpenAI compatibility manifest and display metadata
  .mcp.json                    Codex compatibility MCP configuration
  assets/maitre.png            Existing Maître app icon
  skills/maitre-booking/SKILL.md
```

The package was scaffolded with OpenAI's Plugin Creator. The two MCP
configurations point to the same endpoint; keep them in sync. The World ID and
Maître backends enforce authentication and authorization.

## Demo checks

- Ask "book me a table with World ID": discover Maître tools if needed and use
  live `list_seats` results before asking the user to select a table.
- With browser access denied and Maître MCP available: still use `list_seats`.
- With Maître tools initially hidden: use host discovery before declaring them
  unavailable. If discovery is absent or fails, explain the specific limitation.
- Ask only to connect World ID: do not browse or claim restaurant tables.
- Ask for current tables: use live `list_seats` results; no invented availability.
- Follow a World ID Maître listing: use Maître's MCP to browse and claim.
- Select a table: use the Google account name unless you request an override.
- Verify an unverified account: show the returned link, wait, then retry the
  approved claim. Stop waiting on timeout or error.
- Inspect a submitted claim: report its reference and pending status accurately.
- If a write has an uncertain outcome: check reservations before any retry.
- Cancel a selected reservation: distinguish `cancelled` from
  `cancellation_requested`.

Booking and cancellation checks create or change reservations on the configured
service. Run them only with a table you intend to claim or cancel. Read-only
discovery checks do not verify the complete OAuth and booking flow.

---
date: "2026-11-12T00:00:00+01:00"
draft: false
title: "How my coding agent reached Teams without Graph admin consent"
description: "A coding agent found a first-party Power Platform connector path into Microsoft Teams, without a Power Automate flow or Graph admin consent."
tags: ["ai-agents", "microsoft-teams", "power-platform", "microsoft-graph", "security", "opencode"]

ShareButtons: [linkedin, reddit, x, whatsapp]
showtoc: true
---

I gave my coding agent one sentence:

> *"You are running in WSL on Windows 11. Microsoft Teams is open on the desktop. Find a way to read and send a message to my own self-chat."*

Fifteen minutes and six dead ends later, it posted to my self-chat as me. The route was not a direct Microsoft Graph client. It was the first-party Power Platform Microsoft Teams connector, called directly through its runtime without a Power Automate flow.

The rest of this post is how it happened, what broke, and what it means for Microsoft 365 administrators.

## The wall

When you work in an AI coding agent like [OpenCode](https://opencode.ai) or Claude Code all day, Microsoft Teams is where project context tends to hide. Review requests, deployment notices, incident threads, and quick architectural questions happen in Teams channels or 1:1 chats, while your code, git history, and terminal sit in WSL.

In an enterprise Microsoft 365 tenant, pointing a CLI tool or a third-party [Teams MCP server](https://mcpservers.org/servers/floriscornel/teams-mcp) at Microsoft Graph usually runs into Conditional Access and admin-consent restrictions.

The agent tried six ways in. Three failed in ways worth showing:

| Attempt | Route | What happened |
|---|---|---|
| 1 | Microsoft Graph PowerShell (`Connect-MgGraph`) | Fails from WSL because Web Account Manager requires a native Windows window handle (`A window handle must be configured`). |
| 2 | Graph device-code flow (`14d82eec-…`) | Blocked by Conditional Access with `AADSTS53003` (`BlockedByConditionalAccess`). `AADSTS` is the Microsoft Entra sign-in error prefix; this code means a policy blocked the token.[^1] |
| 3 | Graph browser auth code + PKCE | Passes Conditional Access, then stops at "Need admin approval". Standard users cannot consent to `Chat.ReadWrite` or `ChannelMessage.Send` on unverified or command-line client apps. |

The other three were variations on the same wall: an Azure CLI Graph token without Teams scopes, a retired local WebSocket API, and a deep link that only filled the compose box while the chat was a draft.[^2][^3]

After hitting `AADSTS53003`, the agent wrote a note in its log that I liked:

> *"Error 53003 means a Conditional Access policy blocks the Graph PowerShell app for this account. I won't try other client IDs to get around that."*

Rather than fishing for other client IDs or leaning on UI automation, it checked which first-party Microsoft services my existing `az login` session was already allowed to call.

## The path Microsoft had already approved

Most corporate Microsoft 365 plans include a `Power Automate for Office 365` license so employees can run personal Teams Workflows, approval notifications, and SharePoint automations.

When a Power Automate flow runs a Microsoft Teams action, the Flow Engine (`service.flow.microsoft.com`) executes the workflow, and the API Hub connector runtime (`apihub.azure.com` and `api.powerapps.com`) holds your delegated OAuth connection (`shared_teams`) and forwards requests to Teams and Microsoft Graph.

You do not need a flow to call your own connection. Every connection object in your tenant's default Power Platform environment (`Default-<tenantId>`) has a `testLinks` property pointing to its live connector runtime endpoint, and the `shared_teams` connector has a built-in [Send a Microsoft Graph HTTP request](https://learn.microsoft.com/en-us/connectors/teams/#send-a-microsoft-graph-http-request) operation that forwards delegated Graph calls.[^4] That is the same endpoint the Power Automate portal calls when it populates team and channel dropdowns in the flow designer.

The diagram below compares the route that failed with the route that worked. Path A, at the top, is a standard Graph client or third-party Teams MCP server asking Microsoft Entra for Teams scopes. Path B, below it, is the first-party Power Platform connector runtime.

{{< figure src="images/teams-connector-path.svg" title="Path A: a standard Graph client (top). Path B: the first-party Power Platform connector runtime (bottom)." align="center" >}}

The new route has three moves:

1. Sign in with `az login` in your normal browser. The skill never stores tokens on disk; `az account get-access-token` mints short-lived tokens in memory on each run.
2. Find your `shared_teams` connection in `Default-<tenantId>`, repair it through `getConsentLink` if it is expired, and read the runtime URL from `testLinks[0].requestUri`.
3. Send Graph requests through `POST {runtimeBase}/httprequest` with an `https://apihub.azure.com` bearer token.

The runtime URL handling and the Graph request helper are in [`teams.py`](https://github.com/wolfm89/ms-teams-skill/blob/main/skills/ms-teams/scripts/teams.py).

## The first message

Once I approved the consent link, the agent posted to my self-chat, the private conversation Teams exposes as `48:notes`. The message appeared under my name, not a flow bot's name, and the same connection listed my joined teams and channels across the tenant.

## What surprised me

The connector's high-level post action cannot create channel mentions. It converts `<at>email@example.com</at>` for users, but when I tested `<atChannel>` or `<atTeam>`, or passed a channel display name, it stripped the tags and posted plain text. Sending a raw Graph `chatMessage` payload through `/httprequest` fixed it.[^5] In the skill, you write `@[Ada Lovelace]`, `@[tag:on-call]`, `@[channel:General]`, or `@[team]` inline in HTML. The script resolves each target against the target team's channels, tags, and members, with a tenant user search as a fallback. It rewrites the placeholder, builds the Graph `mentions` array, and checks the response after sending to confirm that Teams attached every mention.

Custom tenant emojis work the same way: the script harvests them while reading and expands `:shortcode:` in HTML.

The digest was the other surprise. A dozen project and deployment channels plus your 1:1 chats can burn tens of thousands of tokens if you feed the agent raw Graph JSON. The `digest` subcommand reads channels you name with `--channel "Team/Channel"` or list in an optional watchlist, plus your active 1:1, group, and meeting chats. It emits a header and one JSON object per post, reply, chat, and chat message, under 1,900 characters per line, with open hints (`mention`, `team`, `question`, `check`). There is no alert or pull-request logic in the script; the watchlist is just a grouped list of team and channel names.

## What this means for admins

Locking down third-party Graph app consent and device-code flows in Microsoft Entra ID does not close this path if the Power Platform environment stays at tenant defaults. Five conditions keep it open:

- **Power Automate license.** Included with standard M365 E3, E5, and Business plans (`Power Automate for Office 365`). Remove the Power Automate service plan for users who do not need workflow automation.
- **Maker access to `Default-<tenantId>`.** Licensed users are automatically assigned the Environment Maker role in the Default environment, and Microsoft does not allow removing that role from the Default environment ([overview](https://learn.microsoft.com/en-us/power-platform/admin/environments-overview), [migration guidance](https://learn.microsoft.com/en-us/power-platform/guidance/white-papers/migrating-from-default-environment)).
- **`shared_teams` allowed by data loss prevention (DLP).** DLP policies decide which Power Platform connectors can be used together and which are blocked. Microsoft Teams is a standard productivity connector; blocking it breaks Teams Workflows and Approvals. Scope the policies and audit active `shared_teams` connections in the Power Platform Admin Center.
- **`az login` tokens for Power Platform.** Engineers use Azure CLI for cloud work. Include Power Platform resource audiences (`service.powerapps.com`, `apihub.azure.com`) in Conditional Access policies. Power Platform service principals are not always covered by the generic "Office 365" CA target group ([Microsoft Learn](https://learn.microsoft.com/en-us/power-platform/guidance/adoption/conditional-access#embedded-flow-authentication-and-conditional-access)).
- **First-party connector consent.** `consent.azure-apim.net` (Azure API Connections) is a pre-authorized first-party Microsoft service principal. Monitor connection creation and consent events in Microsoft Purview and Power Platform activity logs.

## Open-sourcing the skill

I packaged a customer-agnostic, zero-dependency version of the skill in [`wolfm89/ms-teams-skill`](https://github.com/wolfm89/ms-teams-skill) under GPL-3.0-or-later. The first release is [`v0.1.0`](https://github.com/wolfm89/ms-teams-skill/releases/tag/v0.1.0), and the skill is listed on [skills.sh](https://skills.sh/wolfm89/ms-teams-skill).

The script discovers the tenant, user, and `shared_teams` connection from the `az` token and caches them per tenant under `~/.cache/ms-teams/<tenantId>/`, so there is no config file to create. Pass `--tenant <tenantId_or_domain>` (or set `MS_TEAMS_TENANT`) when you work across tenants.

Because an agent using your Teams connection posts under your name, the CLI enforces three guardrails:

1. Every write command (`send`, `reply`, `edit`, `delete`, `react`, `unreact`, `create-chat`, `create-meeting`) runs as a dry-run unless `--send` is passed. A dry-run resolves every `@[...]` mention locally and prints the HTML, plain text, and resolved recipients that would be sent. The skill instructions require the agent to show that preview and wait for your confirmation in chat before adding `--send`. Creating a chat makes it visible to its members, and Teams only lets tenant admins delete chats; creating a meeting with no attendees notifies nobody.
2. `edit` and `delete` fetch the target message first and exit with code `3` if the message author's Entra object ID does not match your user ID.
3. No tokens are written to disk. `az account get-access-token` mints tokens in memory per run. Connection metadata and the team, channel, member, and custom-emoji caches live per tenant under `~/.cache/ms-teams/<tenantId>/` with mode `0600`. The only optional configuration file is `~/.config/ms-teams/watchlist.json`, used by `digest`.

### Installing the skill

You can install `ms-teams` into OpenCode, Claude Code, Codex, GitHub Copilot, Cursor, or Gemini CLI with the [Skills CLI](https://skills.sh):

```bash
npx skills add wolfm89/ms-teams-skill --skill ms-teams -g
```

Or via the GitHub CLI (`gh skill`) or Claude Code's plugin marketplace:

```bash
# GitHub CLI
gh skill install wolfm89/ms-teams-skill ms-teams --scope user --pin v0.1.0

# Inside Claude Code
/plugin marketplace add wolfm89/ms-teams-skill
/plugin install ms-teams@ms-teams-skill
```

After installing, sign in with `az login` and check the setup (adjust the path to the directory your installer used):

```bash
python3 ~/.agents/skills/ms-teams/scripts/teams.py doctor
python3 ~/.agents/skills/ms-teams/scripts/teams.py onboard --smoke
```

Once onboarded, you can ask your coding agent for a morning Teams digest, search a channel for the thread where a decision was made, or preview a channel post with `@mentions` directly from your terminal.

[^1]: [Microsoft Entra authentication and authorization error codes](https://learn.microsoft.com/en-us/entra/identity-platform/reference-error-codes): `AADSTS53003` (`BlockedByConditionalAccess`) indicates that token issuance was blocked by a Conditional Access policy.
[^2]: [Teams `localhost:8124` retirement (`MC1266901`)](https://callstate.app/docs/teams-8124-retired.html): Microsoft retired the local `ws://127.0.0.1:8124` meeting-control WebSocket API on June 30, 2026.
[^3]: [Microsoft Learn: Deep link to a Teams chat](https://learn.microsoft.com/en-us/microsoftteams/platform/concepts/build-and-test/deep-link-teams): `https://teams.microsoft.com/l/chat/0/0?users=…&message=…` only inserts `message` text into the compose box while the chat is in a draft state.
[^4]: [Microsoft Learn: Microsoft Teams Connector: Send a Microsoft Graph HTTP request](https://learn.microsoft.com/en-us/connectors/teams/#send-a-microsoft-graph-http-request): documents the `HttpRequest` operation and its allowed 1st segments (`/teams`, `/me`, `/users`) and 2nd segments (`channels`, `chats`, `installedApps`, `messages`, `pinnedMessages`, `onlineMeetings`).
[^5]: [Microsoft Learn: Send `chatMessage` in a channel or a chat](https://learn.microsoft.com/en-us/graph/api/chatmessage-post?view=graph-rest-1.0): Graph request body format for `<at id="N">` tags and the `mentions` array.

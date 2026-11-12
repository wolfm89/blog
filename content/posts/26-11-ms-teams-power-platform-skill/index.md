---
date: "2026-11-12T00:00:00+01:00"
draft: false
title: "When Microsoft Graph Says 'Need Admin Approval': Reaching Teams via a Power Platform Connector"
tags: ["ai-agents", "microsoft-teams", "power-platform", "microsoft-graph", "security", "opencode"]

ShareButtons: [linkedin, reddit, x, whatsapp]
showtoc: true
---

## A simple prompt in WSL

I started with a one-line prompt to my coding agent:

> *"You are running in WSL on Windows 11. Microsoft Teams is open on the desktop. Find a way to read and send a message to my own self-chat."*

When you work in an AI coding agent like [OpenCode](https://opencode.ai) or Claude Code all day, Microsoft Teams is where project context tends to hide. Pull request review requests, deployment notices, incident threads, and quick architectural questions happen in Teams channels or 1:1 chats, while your code, git history, and terminal sit in WSL.

In an enterprise Microsoft 365 tenant, pointing a CLI tool or a third-party [Teams MCP server](https://mcpservers.org/servers/floriscornel/teams-mcp) at Microsoft Graph usually runs straight into Conditional Access and admin-consent restrictions.

## Six dead ends in fifteen minutes

Exploring the environment read-only, the agent tried six ways to reach Teams from WSL:

| Attempt | Route | What happened |
|---|---|---|
| 1 | Microsoft Graph PowerShell (`Connect-MgGraph`) | Fails from WSL because Web Account Manager (WAM) requires a native Windows window handle (`A window handle must be configured`). |
| 2 | Graph device-code flow (`14d82eec-…`) | Blocked by Microsoft Entra Conditional Access (`AADSTS53003`), which restricts device-code authentication.[^1] |
| 3 | Graph browser auth code + PKCE | Passes Conditional Access, then stops at "Need admin approval". Standard users cannot consent to `Chat.ReadWrite` or `ChannelMessage.Send` on unverified or command-line client apps. |
| 4 | Azure CLI Graph token (`az account get-access-token`) | `az login` works in the browser, but the Azure CLI app registration only holds directory and group scopes, with no Teams chat or channel message scopes. |
| 5 | Teams local WebSocket API (`ws://127.0.0.1:8124`) | Retired by Microsoft (Message Center `MC1266901`). Nothing listens on port `8124`, and the old local API only controlled mute and meeting state.[^2] |
| 6 | `msteams:/` deep link + `SendKeys` | Opening `msteams:/l/chat/0/0?users=…&message=…` and sending `Enter` via PowerShell works once. If the target chat is already open in Teams, Teams ignores the `message=` parameter and `Enter` fires on an empty compose box.[^3] |

After hitting `AADSTS53003` on Attempt 2, the agent wrote a note in its log that I liked:

> *"Error 53003 means a Conditional Access policy blocks the Graph PowerShell app for this account. I won't try other client IDs to get around that."*

Rather than fishing for other client IDs or leaning on UI automation, it checked which first-party Microsoft services my existing `az login` session was already allowed to call.

## The Power Platform connector runtime

Most corporate Microsoft 365 plans (E3, E5, Business Standard, and Business Premium) include a `Power Automate for Office 365` license so employees can run personal Teams Workflows, approval notifications, and SharePoint automations.

When a Power Automate flow runs a Microsoft Teams action, two components do the work:

1. The Flow Engine (`service.flow.microsoft.com`), which executes workflow definitions.
2. The API Hub Connector Runtime (`apihub.azure.com` and `api.powerapps.com`), a first-party Microsoft proxy that holds your delegated OAuth connection (`shared_teams`) and forwards requests to Teams and Microsoft Graph on your behalf.

You do not need a Power Automate flow to call your own connection.

{{< figure src="images/teams-connector-path.svg" title="Path A (standard Graph client apps) vs. Path B (the first-party Power Platform Microsoft Teams connector runtime)" align="center" >}}

Every connection object in your tenant's default Power Platform environment (`Default-<tenantId>`) has a `testLinks` property pointing to its live connector runtime endpoint. That is the same endpoint the Power Automate portal calls in the browser when it populates team and channel dropdowns in the flow designer.

The official Microsoft Teams connector (`shared_teams`) also has a built-in [Send a Microsoft Graph HTTP request](https://learn.microsoft.com/en-us/connectors/teams/#send-a-microsoft-graph-http-request) operation (`POST {runtimeBase}/httprequest`). It forwards delegated `GET`, `POST`, `PATCH`, and `DELETE` requests to Microsoft Graph under `/teams`, `/me`, and `/users` using the connector's first-party Graph token.[^4]

### Wiring it together

Five calls connect the CLI to Teams, each a documented Microsoft endpoint under your own signed-in identity:

1. Sign in with `az login` in your normal browser using the first-party Azure CLI client. The skill never stores tokens on disk; `az account get-access-token` mints short-lived tokens in memory on each run.
2. List your Teams connections in `Default-<tenantId>` by calling `GET /providers/Microsoft.PowerApps/apis/shared_teams/connections` on `https://api.powerapps.com` with a token for `https://service.powerapps.com/`.
3. If an existing `shared_teams` connection is expired or unauthenticated, repair it via `POST .../connections/<id>/getConsentLink`. That call returns a first-party `consent.azure-apim.net` sign-in URL. Approving it in the browser flips the connection status to `Connected` without triggering a third-party "Need admin approval" prompt, because Azure API Connections is a pre-authorized Microsoft service principal.
4. Read the connection's `testLinks[0].requestUri` to get the runtime base URL:
   ```text
   https://<env>.environment.api.powerplatform.com/connectors/runtime/invoke/scalegroups/<region>/shards/<n>/apis/teams/connections/<connectionId>
   ```
5. Send Graph requests through `POST {runtimeBase}/httprequest` using an `https://apihub.azure.com` bearer token from `az`:

```python
def graph(self, method: str, uri: str, body: dict | None = None):
    """Forward a Microsoft Graph request through the shared_teams connector."""
    headers = {
        "Authorization": "Bearer " + az_token("https://apihub.azure.com"),
        "User-Agent": "ms-teams-skill",
        "Uri": uri,
        "Method": method,
    }
    data = b""
    if method != "GET":
        headers["ContentType"] = "application/json"
        data = json.dumps(body or {}, ensure_ascii=False).encode()
    return http("POST", self.base + "/httprequest", headers, data)
```

Once I approved the consent link in my browser, the agent posted a test message to my `48:notes` self-chat. It showed up in Teams under my own name rather than a flow bot's, and the same connection listed my joined teams and channels across the tenant.

## Three API traps we hit building the skill

Turning that first test call into a daily CLI tool exposed three quirks in the Teams and Graph APIs.

### 1. High-level connector actions strip channel `@mentions`

The Teams connector's `PostMessageToConversation` action converts `<at>email@example.com</at>` into a user mention, but when we tested `<atChannel>` or `<atTeam>` tokens, or passed a channel display name, the connector stripped the tags and posted plain text without notifying anyone.

Sending a raw Graph `chatMessage` payload through `/httprequest` to `/v1.0/teams/{teamId}/channels/{channelId}/messages` fixed it.[^5] In the skill, you write `@[Ada Lovelace]`, `@[tag:on-call]`, `@[channel:General]`, or `@[team]` inline in HTML. The script resolves each target against the target team's channels, tags, and members, with a tenant user search as a fallback. It rewrites the placeholder to `<at id="N">Name</at>`, builds the Graph `mentions` array (`aadUser`, `tag`, or `conversation`), and checks the HTTP response after sending to confirm that Teams attached every mention. Standard Unicode emojis work directly; custom tenant emojis harvested while reading can be written as `:shortcode:` and are expanded to `<customemoji>` tags.

### 2. Channel replies break `createdDateTime` paging

When you fetch channel messages via `/teams/{id}/channels/{id}/messages?$expand=replies`, Graph orders the top-level posts by `lastModifiedDateTime` instead of `createdDateTime`.[^6] Whenever someone replies to a week-old thread today, that thread moves to the top of page 1, above newer threads that have no replies.

If a script stops paging as soon as it sees a root message whose `createdDateTime` is older than `--since`, it misses active threads. The script keeps paging until an entire page has `lastModifiedDateTime < since`, then keeps any older thread that got replies inside the window.

### 3. Keeping morning digests inside an agent's context budget

Asking an agent to summarize a dozen project and deployment channels plus your 1:1 chats can burn tens of thousands of tokens if you feed it raw Graph JSON.

The `digest` subcommand (`teams.py digest --since last-workday`) reads the channels you name with `--channel "Team/Channel"` or list in an optional watchlist at `~/.config/ms-teams/watchlist.json`, plus your active 1:1, group, and meeting chats. It emits a header and one JSON object per post, reply, chat, and chat message, with each line capped under 1,900 characters so the agent's file-read tool can read the whole digest in one call without truncation.

The records also carry deterministic open hints (`mention`, `team`, `question`, `check`), so the agent can sort what needs an answer from what is only background. There is no alert or pull-request logic in the script; the watchlist is just a grouped list of team and channel names.

## Security and governance

This path stays inside the signed-in user's permissions. The connector runs under your own delegated identity, so it cannot read a channel you have not joined, open another user's private chats, or perform any action you cannot already do in the Teams app. Its `HttpRequest` passthrough is also narrower than raw Graph: the connector only accepts URI paths starting with `/v1.0/` or `/beta/` followed by `teams`, `me`, or `users`, and only for Teams resources such as `channels`, `chats`, `messages`, `installedApps`, `pinnedMessages`, and `onlineMeetings`.[^4] Directory, mail, or SharePoint paths return HTTP 400.

Cloud architects and Microsoft 365 administrators have to reason about both control planes: locking down third-party Graph app consent and device-code flows in Microsoft Entra ID does not restrict first-party Power Platform connectors if the Power Platform environment stays at tenant defaults.

All five conditions below must hold for this route to work:

| # | Condition required | Why it is usually open by default | How administrators can govern or restrict it |
|---|---|---|---|
| 1 | Power Automate license | Included with standard M365 E3, E5, and Business plans (`Power Automate for Office 365`). | Remove the Power Automate service plan for users who do not need workflow automation. |
| 2 | Maker access to `Default-<tenantId>` | Licensed users are automatically assigned the Environment Maker role in the Default environment, and Microsoft does not allow removing that role from the Default environment.[^7] | Enable Default environment routing and apply a strict DLP data policy to the Default environment to control which connectors makers can use there. |
| 3 | `shared_teams` allowed by DLP | Microsoft Teams is a standard productivity connector; blocking it breaks Teams Workflows and Approvals. | Scope Power Platform Data Loss Prevention (DLP) policies and audit active `shared_teams` connections in the Power Platform Admin Center. |
| 4 | `az login` tokens for Power Platform | Engineers use Azure CLI for cloud work. | Include Power Platform resource audiences (`service.powerapps.com`, `apihub.azure.com`) in Conditional Access policies. Power Platform service principals are not always covered by the generic "Office 365" CA target group.[^8] |
| 5 | First-party connector consent | `consent.azure-apim.net` (Azure API Connections) is a pre-authorized first-party Microsoft service principal. | Monitor connection creation and consent events in Microsoft Purview and Power Platform activity logs. |

## Open-sourcing `ms-teams-skill`

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
[^6]: [Microsoft Learn: List channel messages](https://learn.microsoft.com/en-us/graph/api/channel-list-messages?view=graph-rest-1.0): responses from `GET /teams/{team-id}/channels/{channel-id}/messages` are sorted by `lastModifiedDateTime` of the entire reply chain and only support `$top` and `$expand`.
[^7]: [Microsoft Learn: Power Platform environments overview](https://learn.microsoft.com/en-us/power-platform/admin/environments-overview) and [Migrating apps and flows from the default environment](https://learn.microsoft.com/en-us/power-platform/guidance/white-papers/migrating-from-default-environment): all licensed users receive the `Environment Maker` role in the Default environment, and that role cannot be removed from the Default environment.
[^8]: [Microsoft Learn: Configure identity and access management: Conditional Access for Power Platform](https://learn.microsoft.com/en-us/power-platform/guidance/adoption/conditional-access#embedded-flow-authentication-and-conditional-access): notes that Microsoft Flow Service (`7df0a125-d3be-4c96-aa54-591f83ff541c`) is not included in the generic Office 365 Conditional Access application target.

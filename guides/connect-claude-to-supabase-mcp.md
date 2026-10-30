---
title: "How to connect Claude to a Supabase project with MCP (read only, one project)"
description: "Give Claude read-only access to a single Supabase project through Supabase's hosted MCP server, using Claude's web and desktop apps or Claude Code."
nav_order: 2
---

# How to connect Claude to a Supabase project with MCP (read only, one project)

## Summary

Supabase runs a hosted Model Context Protocol (MCP) server that lets AI assistants like Claude list tables, run SQL, read logs, and search the Supabase docs on your behalf. Out of the box, that server can reach every project in the organization you sign in to, and it can write as well as read. This guide shows you how to lock it down to one project, in read-only mode, with only the tool groups you need, and then connect it to Claude's web and desktop apps or to Claude Code. Plan on about 15 minutes, most of it spent signing in and testing.

## Before you start

- **A Supabase account** with access to the project you want Claude to read. You'll sign in through your browser during setup.
- **The project's reference ID (project ref).** This is the short ID that appears in the project's dashboard URL and in its settings. In this guide it's shown as `<project-ref>`.
- **A Claude plan that supports connectors.** Custom connectors work on Free, Pro, Max, Team, and Enterprise plans. Free plans are limited to one custom connector.
- **On Team or Enterprise plans, an organization Owner.** Only Owners (and on Enterprise, people with the right custom role) can add a custom connector for the organization. Members then connect to it individually.
- **For the Claude Code path,** Claude Code installed and a terminal open in your project folder.
- **A decision about data.** If this is a production project, confirm that reading it with an AI assistant is allowed under your data policies. Supabase's own guidance is to connect production only when the task truly needs production evidence.

## Step 1: Find your project ref

1. Sign in to the Supabase dashboard and open the project you want to connect.
2. Open the project's settings and copy the project ref (also called the reference ID).

![Supabase project settings page showing where to find the project ref]({{ site.baseurl }}/assets/screenshots/supabase-01.png)

## Step 2: Build your scoped server URL

The hosted server lives at `https://mcp.supabase.com/mcp`. You narrow what it can do by adding query parameters to that URL:

| Parameter | What it does |
| --- | --- |
| `project_ref=<project-ref>` | Limits the server to one project. Account-level tools such as listing or creating projects are turned off. |
| `read_only=true` | Runs every SQL query as a read-only Postgres user. |
| `features=database,docs` | Turns on only the tool groups you list. |

Combine them into one URL:

```text
https://mcp.supabase.com/mcp?project_ref=<project-ref>&read_only=true&features=database,docs
```

For a read-only reporting or troubleshooting use case, `database,docs` is a good starting set. Add `debugging` if you also want Claude to query logs and read Supabase's security and performance advisors. Leave out `development`, `functions`, and `branching` unless you have a clear reason to include them.

![Supabase docs MCP setup panel with read only on and only the database and docs feature groups]({{ site.baseurl }}/assets/screenshots/supabase-02.png)

## Step 3a: Connect in Claude (web or desktop app)

Supabase is listed as a connector in Claude's directory. If you want the scoped URL from Step 2, add it as a custom connector instead, so you control the exact URL Claude uses.

**On Pro and Max plans:**

1. In Claude, open **Customize > Connectors**.
2. Choose the option to add a custom connector.
3. Give it a clear name, such as "Supabase (read only, reporting project)".
4. Paste the scoped URL from Step 2 and continue.
5. Review the sign-in settings Claude detects, then finish adding the connector.
6. Select **Connect**. A browser window opens so you can sign in to Supabase and approve access. Pick the organization that contains your project.

**On Team and Enterprise plans:** an Owner adds the connector once under **Organization settings > Connectors** with the same name and URL. Members then find it under **Customize > Connectors** and select **Connect** to sign in with their own Supabase account.

Custom connectors work the same way in the Claude desktop app, because Claude reaches the server from Anthropic's cloud rather than from your computer. The hosted Supabase server is already public, so there's nothing to open in your firewall.

## Step 3b: Connect in Claude Code

From your project folder, add the server with the scoped URL. Quote the URL so your shell doesn't treat the `&` characters as commands:

```bash
claude mcp add --scope project --transport http supabase \
  "https://mcp.supabase.com/mcp?project_ref=<project-ref>&read_only=true&features=database,docs"
```

`--scope project` writes the server to a `.mcp.json` file in the project root, so teammates who use the same repository get the same locked-down setup. Use `--scope local` instead if you want it to stay private to you.

Then sign in. Start Claude Code in a regular terminal, run `/mcp`, choose the `supabase` server, and select **Authenticate**. You can also run `claude mcp login supabase` straight from your shell.

**For automated or CI use only:** if there's no browser for OAuth, Supabase supports a personal access token passed as a header. Create a token scoped to this one project with only the permissions your tools need, store it in an environment variable, and add:

```bash
claude mcp add --transport http supabase \
  "https://mcp.supabase.com/mcp?project_ref=<project-ref>&read_only=true&features=database,docs" \
  --header "Authorization: Bearer ${SUPABASE_ACCESS_TOKEN}"
```

Never paste the token value itself into a chat, a ticket, or a committed file.

## Step 4: Test the connection

Ask Claude something that only works if the connection is live, such as: "What tables are in the database? Use the Supabase MCP tools."

Then confirm read-only mode is doing its job. Ask Claude to create a throwaway table in a test project. The query should fail because it runs as a read-only Postgres user.

## What to watch for

- **The default URL is wide open.** Without `project_ref`, the server can see every project in the organization you approved and use account tools such as creating or pausing projects. Without `read_only=true`, it can run writes and migrations. Always check the URL before you save it.
- **Pick the right organization at sign-in.** The approval screen asks which organization to grant. If you choose the wrong one, the project ref won't resolve.
- **Prompt injection is real.** If your tables hold text that customers or the public typed, that text could contain instructions aimed at the AI. Supabase wraps SQL results to discourage Claude from following them, but says this isn't foolproof. Keep tool-call approval turned on and read what Claude is about to run.
- **Not for end users.** The server acts with your developer permissions. Supabase is clear that it's an internal tool, not something to hand to customers.
- **Some tools are paid or off by default.** Branching tools need a paid plan, and the Storage group is off unless you turn it on.
- **Restart if tools don't appear.** Some clients only pick up the full tool list after a restart following sign-in.

## Controlling access

Think of access in three layers, and tighten each one:

1. **Which project:** `project_ref` limits Claude to one project and turns off account-level tools.
2. **What kind of query:** `read_only=true` runs SQL as a read-only Postgres user, so inserts, updates, deletes, and schema changes fail at the database.
3. **Which tools:** `features` hides whole tool groups. If Claude doesn't need Edge Functions or branching, don't expose them.

On top of that, keep your client's per-call approval on for interactive work, prefer a development branch or staging project over production, and remove the connector (or run `claude mcp remove supabase`) when the work is done. On Team and Enterprise plans, the Owner controls which connectors exist, and each member signs in with their own Supabase account, so Claude only sees what that person can see.

## Read only vs. read-write: a comparison

| | Read only (`read_only=true`, `project_ref` set) | Read-write (no `read_only`, no `project_ref`) |
| --- | --- | --- |
| Projects reachable | One | Every project in the approved organization |
| SQL writes and migrations | Blocked at the database | Allowed |
| Account tools (create, pause, restore projects) | Turned off | Available |
| Best for | Reporting, troubleshooting, schema questions, audits | Hands-on development in a non-production project |
| Risk if Claude is misled by injected text | Data could be read, but not changed | Data could be read, changed, or deleted |
| Recommended for production | Yes, if production access is needed at all | No |

## FAQ

**Do I need a personal access token?**
No, not for normal use. The hosted server uses OAuth with dynamic client registration, so you sign in through your browser. Tokens are only for CI or other places where a browser sign-in isn't possible.

**Can I connect two projects?**
Yes. Add two connectors (or two Claude Code servers) with different names, each with its own `project_ref`. Keep each one read-only unless you have a reason not to.

**Does read-only mode stop Claude from seeing sensitive columns?**
No. Read-only stops changes, not reads. If a table holds data Claude shouldn't see, connect to a staging copy instead, or leave that project out entirely.

**Why does Claude say it can't list my projects?**
That's expected. Setting `project_ref` turns off the account-level tools, including project listing.

**How do I remove access later?**
In Claude, disconnect or remove the connector under Customize > Connectors. In Claude Code, run `claude mcp logout supabase` to revoke the stored token where supported, then `claude mcp remove supabase`.

## Sources

- Supabase: [Supabase MCP Server](https://supabase.com/docs/guides/getting-started/mcp)
- Supabase: [MCP server repository](https://github.com/supabase/mcp)
- Anthropic Help Center: [Custom connectors using remote MCP](https://support.claude.com/en/articles/11175166)
- Anthropic Help Center: [Using connectors in Claude](https://support.claude.com/en/articles/11176164)
- Anthropic: [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)

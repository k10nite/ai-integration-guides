# Project: AI Integration Guides (public portfolio site)

This folder is a public GitHub Pages site holding two technical how-to guides used as work samples in Upwork proposals. Neil writes the guide content; help him write, edit, fact check and format.

## Guides
1. guides/connect-claude-to-supabase-mcp.md: connect Claude to Supabase with MCP, read only, scoped to one project.
2. guides/connect-claude-to-microsoft-365-mailbox.md: connect Claude to a Microsoft 365 mailbox.
Starting drafts are in drafts/ (excluded from the site).

## Each guide follows this shape
Short summary, what you need before you start, numbered steps with screenshots, what to watch for, how to control what the AI can access, a comparison table of setup options, an FAQ with 5 questions, and a sources list of official docs. Plain American English for a technical but non-developer IT leader.

## Hard rules (the site is public and linked from Upwork)
- No email addresses, phone numbers, LinkedIn or social links, personal website, Calendly, or any contact section or author bio anywhere.
- No client names, client domains, real tenant IDs, real Supabase project refs, API keys, tokens or secrets. Use placeholders like contoso.onmicrosoft.com and <project-ref>.
- Screenshots go in assets/screenshots/ and must come from demo or sandbox accounts only. Never use a client or production project. Check every image for emails, avatars, IDs and keys before committing.
- Verify technical claims against official docs (Supabase, Anthropic, Microsoft Learn). If something can't be confirmed, flag it instead of guessing.
- Don't create the GitHub repo, push, or make anything public unless Neil explicitly asks.

## Before every commit
Run: grep -rniE "@[a-z0-9-]+\.(com|co|net|io)|linkedin|twitter|x\.com/|facebook|phone|perfectted|made beverages|wvcg" --include=*.md --include=*.yml . | grep -v CLAUDE.md
It should return nothing except placeholder addresses on contoso.onmicrosoft.com.

# Project: AI Integration Guides (public portfolio site)

This folder is a public GitHub Pages site holding two technical how-to guides used as work samples in Upwork proposals. Neil writes the guide content; help him write, edit, fact check and format.

## Guides
1. guides/connect-claude-to-supabase-mcp.md: connect Claude to Supabase with MCP, read only, scoped to one project.
2. drafts/published-connect-claude-to-microsoft-365-mailbox.md: connect Claude to a Microsoft 365 mailbox. Parked in drafts/ for now, so it is not on the live site.
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

## Open task: finish the Supabase screenshots
The Microsoft 365 guide is parked in drafts/published-connect-claude-to-microsoft-365-mailbox.md and is hidden from the live site. The focus is the Supabase guide, which is the only live guide.
The Supabase guide already shows supabase-01 to supabase-04. The remaining Supabase shots are:
- supabase-05: the claude mcp add command in a terminal, with the "Added" confirmation line.
- supabase-06: the Claude Code /mcp panel showing the supabase server as connected.
- supabase-07: Claude listing the project's tables in response to the test prompt.
- supabase-08: Claude reporting that a write query was refused in read-only mode.
The full shot list with placement is in drafts/connect-claude-to-supabase-mcp.md (the lines starting "SCREENSHOT:").
When Neil starts a session, help him:
1. Take each screenshot from demo or sandbox accounts only (a fresh "Demo" Supabase org and project, a Microsoft 365 sandbox tenant). Never use a client or production project.
2. Check each image for emails, names, avatars, project refs, tenant or client IDs and keys, and cover anything private with solid boxes (not blur).
3. Save the images to assets/screenshots/ using the names in drafts/ (supabase-01.png and so on).
4. Put each image back in the guide at the right step with clear alt text, as ![description]({{ site.baseurl }}/assets/screenshots/name.png).
5. Fix any button or field names in the text that don't match what the screenshots show, and resolve the "Note to verify" items from drafts/.
6. Run the pre-commit check, then commit and push. The site rebuilds in about a minute.

---
name: reddit-poster
description: Draft and publish Reddit posts that read human (lowercase, story-first) using the cskwork/reddit-mcp tools, with flair lookup, dry-run, and Reddit Responsible Builder Policy compliance.
when_to_use: User asks to "post on Reddit", "share this on r/X", "advertise on Reddit", or invokes /reddit-poster. Also when editing or deleting their own posts.
allowed-tools: Bash(uv *) Bash(reddit-post *) Read Write
---

Wrap the `cskwork/reddit-mcp` toolkit so Claude can take a project, repo, or idea and publish a Reddit post that doesn't read like marketing copy. Available as a CLI (`reddit-post`) or as MCP tools (`create_post`, `edit_post`, `delete_post`, `list_flairs`, `get_post`, `search_reddit`).

Repo: <https://github.com/cskwork/reddit-mcp>

## Verify once per session

```bash
uv --version       # 0.4+ recommended
cd <path-to-reddit-mcp> && uv run reddit-post --help
```

If credentials missing, the CLI raises with the exact env vars to set. Defaults: env vars first, then `~/.claude.json`'s `mcpServers.reddit.env` fallback.

## The four-step flow

Every Reddit post follows this loop. Don't skip steps.

1. **Discover** — list flairs and read tone of recent posts in the target sub.
2. **Draft** — write a human-style body, show it to the user, iterate.
3. **Dry-run** — confirm flair resolution and length before going live.
4. **Post** — get explicit user approval, then `create_post`.

### Step 1 — discover

Pick exactly one subreddit. Cross-posting identical content violates Reddit Responsible Builder Policy and the MCP itself blocks it.

```bash
uv run reddit-post flairs <subreddit>
```

If the sub requires flair, you'll see them. If it has none, the CLI says so. Pick a flair that matches the post type (Showcase / Project / Discussion / Help — varies by sub).

Optionally skim the sub's recent top posts to match register:

```bash
# via the MCP server, when wired in
search_reddit(query="*", subreddit="<name>", limit=10)
```

### Step 2 — draft (the human-style rules)

The post must read like a person sharing something they built, not a landing page. Hard rules:

- **Open with a personal moment, not a feature list.** "got tired of jumping between two terminals…" beats "Single skill, three subcommands:". Lead with the pain or the trigger.
- **Default to lowercase, casual sentences.** Reserve capitals for proper nouns and code identifiers.
- **No TL;DR, no heavy bold, no emoji, no marketing adjectives.** Skip "powerful", "blazing", "seamless", "easy-to-use".
- **Bullet lists are allowed but sparing.** One tight list of two or three items. Never the whole post.
- **Acknowledge limitations honestly.** Edge cases and known bugs in line, not buried at the bottom.
- **End with a low-key invitation.** "happy to take questions or feedback" — not "smash that upvote".
- **Code references inline with backticks.** No code blocks unless the snippet is non-trivial.
- **Keep title plain.** Reddit titles are immutable post-submit; a typo or extra prefix means delete + repost (which risks spam detection). Triple-check before submitting.

### Step 3 — dry-run

Always dry-run. Confirms flair resolves, shows the body byte count, surfaces title issues.

```bash
uv run reddit-post post \
  --subreddit <name> \
  --title "<title>" \
  --body-file draft.md \
  --flair "<Flair Text>" \
  --dry-run
```

If flair resolution fails, the error lists every available flair. Pick one and retry.

### Step 4 — post (with explicit user approval)

Show the user the resolved plan from the dry-run, then ask for explicit go-ahead. **Do not auto-submit.** Posting is an irreversible external action.

```bash
uv run reddit-post post \
  --subreddit <name> \
  --title "<title>" \
  --body-file draft.md \
  --flair "<Flair Text>"
```

Returns `{id, url, title, subreddit, flair_id, flair_text}`. Verify by re-fetching:

```bash
uv run reddit-post get <url>
```

Confirm `link_flair_text` matches what you intended — that's the only way to be sure flair landed.

## Editing and deleting

Reddit allows editing the **body** of self posts, not the title.

```bash
uv run reddit-post edit <url> --body-file new_body.md
```

Title-only changes require **delete + repost**. Get explicit user approval before either:

```bash
uv run reddit-post delete <url>
uv run reddit-post post --subreddit <name> --title <new> ...
```

Risks of delete + repost:

- The original URL dies (broken inbound links from elsewhere).
- Reddit may flag the repost as spam if the body is identical and timing is close. If the original had engagement (>1 score, any comments), warn the user before deleting.

## Disclosure (Reddit Responsible Builder Policy)

Reddit's [Responsible Builder Policy](https://support.reddithelp.com/hc/en-us/articles/42728983564564-Responsible-Builder-Policy) requires bot-generated content to clearly disclose its automated nature.

- "Bot-generated" includes posts where an LLM drafted the body, even if the human approved.
- The policy-safe form is a one-line italic disclosure at the bottom: *posted via my own Reddit MCP — https://github.com/cskwork/reddit-mcp*
- If the user asks to omit the disclosure, **flag the policy conflict once** ("technically this triggers the bot-disclosure rule and risks moderator action"), then comply with the user's call. The user owns the account and the consequences.
- Never silently drop the disclosure to make a post look more organic. Always surface the choice.

## What not to do

- **Don't cross-post.** One subreddit per piece of content. Different sub? Rewrite the body for that audience.
- **Don't manipulate votes or karma.** Don't ask people to upvote, don't post the same thing from multiple accounts, don't brigade.
- **Don't DM users.** The MCP doesn't expose DM tools and shouldn't.
- **Don't post without the user's go-ahead on the live submit step.** Drafts are free; submissions cost trust.
- **Don't paste the user's secrets, internal docs, or unreleased code into a public post.** Re-confirm with the user if any content looks sensitive.

## When the slash command is invoked

If the user types `/reddit-poster` with a description of what they want posted, infer the subreddit from context (their previous request, the project's audience). If unclear, ask once. Then run the four-step flow above and stop at step 4 for explicit approval.

If the user types `/reddit-poster edit <url> ...` or `/reddit-poster delete <url>`, route to the edit/delete flow above.

## Output discipline

- Don't dump raw CLI output. Summarize: post URL, title, flair, score/comments after verification.
- For drafts, show the rendered body in a fenced markdown block so the user can preview formatting.
- For errors (flair not found, auth missing, spam detection), show the actionable fix, not the stack trace.

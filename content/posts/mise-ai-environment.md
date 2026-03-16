---
title: "mise en place for the AI Age"
date: 2026-03-16
draft: false
ai: true
tags: ["AI", "mise", "Bash", "Workflow", "Automation", "Tools"]
description: "Agentic AI wastes a surprising number of tokens reconstructing commands it already knows how to run. mise is the prep work that fixes that, and the name is not a coincidence."
image: "images/posts/mise-ai-environment.png"
---

Before a restaurant kitchen opens for the evening, the chef prepares. Ingredients are chopped, sauces are started, tools are laid out in reach. The French call this *mise en place*: everything in its place. The service goes smoothly not because the chef is faster in the moment, but because the thinking happened in advance.

There's a tool for developers called [mise](https://mise.jdx.dev/). It's a version manager and task runner, and it happens to take its name from the same concept. I don't think that's an accident: it's a tool that exists to have everything ready before the work starts. As AI agents have become part of my daily workflow, the metaphor has gotten more literal.

## The hidden tax

Agentic AI has a cost that doesn't show up in demos: token consumption. Every round-trip in an agentic loop costs tokens across multiple categories. Input tokens cover the system prompt, tool definitions, the entire conversation history, and the new message. Output tokens cover everything the model generates: reasoning, prose, and tool call invocations.

The pricing asymmetry matters. On current Anthropic models, [output tokens cost roughly three to five times more than input tokens](https://www.anthropic.com/pricing). This is not a detail: it's the crux of why the way you structure agent workflows has a direct economic consequence.

When an agent has to construct a complex shell command from scratch, it generates that command as output. A realistic `curl` call with bearer authentication, content-type headers, and a `jq` filter to extract the fields you care about might run to a hundred or more output tokens. The equivalent named command, `mise run jira:get PROJ-1234`, is a handful of tokens. If that operation happens ten times in a session, the difference is substantial. Across a long session with many tool calls, this adds up in a way that is measurable.

Worth noting: [Anthropic's own token counting documentation](https://docs.anthropic.com/en/docs/build-with-claude/token-counting) shows that even a single simple tool definition adds roughly 390 tokens to an otherwise 14-token message. The system prompt, tool schemas, and conversation history are all input tokens, and they accumulate.

## The retry loop

There's a second cost, less obvious but arguably larger in practice. Dynamically constructed commands fail. The wrong flag, a missing quote, an incorrect auth format: the command runs, returns an error, the agent reads the error, reasons about the correction, and generates a new attempt. Each cycle is a full round-trip: more output tokens, more input tokens (the error result enters the context), more cost, more latency.

Complex agentic runs routinely consume over 100,000 tokens per task, much of it in tool-calling loops. Reducing per-call overhead compounds across those loops. Anthropic has noted publicly that tool format design directly affects model efficiency, to the point where optimizing the tools matters more than optimizing the prompt itself.

## How an agent figures out a command

Here is what actually happens when you ask an agent to do something that involves an external system it hasn't used before. Say you ask it to fetch a Jira issue.

Jira exposes a REST API. The agent knows this, or can find it out. What it doesn't know is the exact endpoint structure, the authentication header format, or which fields in the response you care about. So it reasons through it: looks up the API, constructs a curl command, runs it, gets back a wall of JSON, writes a jq filter to extract the relevant parts, runs it again, adjusts. Two or three round-trips, each one generating output tokens and feeding results back as input tokens. Eventually it has something that works.

That discovery process is genuinely useful. The agent figured out the right incantation, validated it against the real API, and handled the edge cases. The problem is that this work evaporates at the end of the session. Next time you ask, the agent starts from zero and does it all again.

There are two ways to capture that work for future sessions. You can write it into a memory file, which the agent loads on every session whether that integration is needed or not. Or you can write it into a skill, a reusable instruction set the agent loads when relevant. Both are better than nothing, but both pay with context: a skill that describes a non-trivial API integration in enough detail to be useful ends up being a small manual, and that manual enters the context every time the skill is invoked.

A script has none of that overhead. It lives outside the context entirely. The agent calls it with a handful of tokens and gets the result back. So the question becomes: what's the right place to put it?

## The obvious alternative, and why it's not enough

The obvious answer is shell scripts. Write the curl command once, put it in a `scripts/` folder, call it a day. It's a reasonable instinct, and for a solo project with a single machine it mostly works. But it comes with a set of problems that compound as the project grows, the team changes, or you simply switch laptops.

Environment variables need to be set somewhere. In practice this means a `.env` file that gets sourced manually, or entries in `~/.zshrc` that only exist on your machine, or a `README` section that says "make sure you have these set" and is always slightly out of date. Secrets leak into git histories more often than anyone admits. The AI, for its part, has no idea which variables are available and which aren't unless you tell it every time.

Tool versions drift. The `jq` that ships with macOS is older than the one on Linux CI. The Node version that works on your laptop is not necessarily the one your colleague has installed. Scripts that run locally silently fail elsewhere, and the failure looks like a bug in the script rather than a version mismatch.

And discoverability disappears entirely. A folder of shell scripts has no index, no descriptions, no dependency declarations. The AI can `ls scripts/` and guess, or it can ask you, or it can read every script until it finds the one it needs. None of these are efficient.

What you actually want is a system where tool versions are declared and automatically installed, secrets are sourced from the environment in a consistent way, and every workflow has a name, a description, and a place to live. Building that by hand is possible. Or you can use something that already solved it.

## What mise is

[mise](https://mise.jdx.dev/) (pronounced "meez") is a polyglot version manager and task runner. It replaces the fragmented ecosystem of `nvm`, `rbenv`, `pyenv`, `asdf`, and similar tools, plus Make, with a single unified interface. Installation is a one-liner:

```bash
curl https://mise.run | sh
```

Or via Homebrew on macOS:

```bash
brew install mise
```

Once installed, a `mise.toml` at the root of a project handles three concerns in one file:

```toml
[tools]
node = "22"
ruby = "4"
java = "17"
flutter = "latest"
tuist = "4"
"spm:synesthesia-it/Murray" = "latest"

[env]
JIRA_BASE_URL = "https://jira.yourcompany.com"
JIRA_PAT = "{{ env.JIRA_PAT }}"

[tasks.jira-get]
description = "Fetch a Jira issue by key"
run = """
curl -s \
  -H "Authorization: Bearer $JIRA_PAT" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/2/issue/$1" \
  | jq '{key:.key, summary:.fields.summary, status:.fields.status.name}'
"""
```

`[tools]` pins every runtime and CLI tool at a specific version, installed consistently on every machine via `mise install`. Versions support semver: `"22"` installs the latest patch of Node 22, `"2.x"` tracks minor releases, `"latest"` follows the tip. Backends extend well beyond the default registry: `spm:` installs Swift packages directly from GitHub via Swift Package Manager, `aqua:` taps into the Aqua registry for a wide catalogue of CLI tools. If a tool exists somewhere, mise can probably install it. `[env]` makes secrets available as environment variables, sourced from the shell environment where they live safely: a keychain, a secrets manager, or a gitignored local file. They never appear in a prompt. `[tasks]` gives every workflow a name and a description.

Inline tasks in the TOML work for simple cases. For anything more complex, mise discovers tasks from a dedicated folder: `.mise/tasks/`, `mise/tasks/`, and a few other conventional paths. Each file is an executable script with a structured header:

```bash
#!/usr/bin/env bash
# [MISE]
# description = "Fetch a Jira issue by key"
# alias = "jg"
# depends = ["check-env"]

set -euo pipefail

ISSUE_KEY="${1:?Usage: mise run jira:get ISSUE-KEY}"

curl -s \
  -H "Authorization: Bearer $JIRA_PAT" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/2/issue/$ISSUE_KEY" \
  | jq '{
      key: .key,
      summary: .fields.summary,
      status: .fields.status.name,
      description: .fields.description
    }'
```

This file lives at `.mise/tasks/jira/get` and is called as `mise run jira:get PROJ-1234`. The subdirectory becomes a namespace, using `:` as separator. `mise tasks` lists everything with descriptions. `depends` declares prerequisites that run automatically first.

The `# [MISE]` block syntax (with a space after `#`) is formatter-safe: most linters and autoformatters that silently add spaces to comment markers won't corrupt it.

AI models write bash fluently and without prompting. The same model that writes your application code writes your tooling scripts, and the output is readable and correct. Makefiles, by comparison, have a syntax full of footguns: meaningful whitespace, automatic variables like `$@` and `$<`, `.PHONY` declarations. mise tasks are just scripts. The barrier is lower, and the AI clears it reliably.

## A concrete example: Jira on-premise

The [official Atlassian MCP server](https://support.atlassian.com/atlassian-rovo-mcp-server/docs/getting-started-with-the-atlassian-remote-mcp-server/) (Rovo) connects AI tools to Jira, but it only supports Jira Cloud. Teams running Jira Server or Data Center on-premise are currently excluded: there is an open feature request ([JRASERVER-78874](https://jira.atlassian.com/browse/JRASERVER-78874)) with no public roadmap commitment.

Jira's REST API, however, works identically on every version. Personal Access Tokens for authentication have been supported since Server 8.14. The task above is the entire integration: a single curl call, a PAT in the environment, a jq filter. No MCP, no plugin, no additional dependency.

This pattern generalizes. Every REST API is, at its core, a curl command. There is no integration that strictly requires a dedicated tool: there is only the question of whether the API exists, and it almost always does.

## Connecting AI to the kitchen

The last piece is telling the AI how to use what you've prepared. In Claude Code, skills are short markdown files that give the agent reusable instructions. Here's one that connects Claude to the Jira task:

```markdown
---
name: jira-context
description: Fetch a Jira issue to use as context before starting work
---

When working on something that references a Jira issue, fetch it before writing any code:

    mise run jira:get {{ISSUE_KEY}}

Use the output to understand the scope and acceptance criteria.
Do not ask the user to paste Jira content manually.
```

The AI reads this, knows exactly what command to run, and calls it the same way every time: a pre-tested script, no reconstruction, no retry. The PAT never enters the conversation. The URL is never reconstructed from memory.

Compare that to what the skill would contain without mise. To give the agent equivalent instructions, it would need to encode the endpoint structure, the authentication header, the environment variable names, and the jq fields to extract:

```markdown
---
name: jira-context
description: Fetch a Jira issue to use as context before starting work
---

Fetch Jira issues using the REST API:

    curl -s \
      -H "Authorization: Bearer $JIRA_PAT" \
      -H "Accept: application/json" \
      "$JIRA_BASE_URL/rest/api/2/issue/ISSUE-KEY" \
      | jq '{key:.key, summary:.fields.summary, status:.fields.status.name, description:.fields.description}'

JIRA_BASE_URL is https://jira.yourcompany.com. JIRA_PAT is a Personal Access Token from your profile settings.
If the request returns a 401, the token may have expired. Use key, summary, status, and description from the response.
```

That skill enters the context on every invocation. If the API changes, or the jq filter needs an extra field, you update the skill and pay those tokens again on every load. The mise version has one meaningful line. Everything else is in the script, outside the context entirely.

This is the division the metaphor describes: the chef doesn't prepare the kitchen and also cook at the same time. Mise en place happens before service. The AI focuses on judgment and code generation; mise provides a tested, consistent execution layer for everything external.

## What this does not fix

To be honest about the trade-off: the savings described here are primarily in **output tokens** and failed retry loops. They are real, but they are not the whole picture.

Tool result tokens are unaffected. If `mise run jira:get PROJ-1234` returns a 2,000-token JSON payload, those 2,000 tokens enter the context as input on the next turn, the same as if the AI had run the full curl directly. What you invoked has no bearing on what comes back.

If you expose mise tasks as formal tool schemas with parameter descriptions, those schemas add input tokens to every API call in the session, though [Anthropic's prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) (which discounts repeated context at roughly 10% of normal cost) significantly reduces the amortized overhead for long sessions.

The net saving is most meaningful for commands that are complex, called frequently, and prone to construction errors when generated dynamically. For simple one-liners, the overhead of having the tool definition at all may outweigh the benefit.

## Without the AI

Everything here works without any AI in the loop. `mise run jira:get PROJ-1234` works in a terminal, in CI, in a shell script, called by you or by an automated process. The commands you define for an agent are the same commands you use yourself, because they're just commands.

That's what makes this worth doing independent of AI efficiency arguments. A mise configuration that clearly names every external dependency, pins every tool version, and expresses every workflow as a documented command is a useful thing to have regardless of what any AI model can do with it. The token savings are a side effect of being well-organized.

The kitchen is already set up. The AI just gets to eat without making a mess.

## The recipe

The Jira example in this post is deliberately simple: one curl call, one jq filter. Real integrations are messier. Pagination, error handling, multiple endpoints, conditional logic. The point was to show the pattern without burying it in bash. For anything non-trivial, the workflow scales the same way.

1. **Ask the AI to write the script.** Give it the goal, not the implementation. "Write a bash script that fetches all open Jira issues assigned to me in the current sprint and formats them as a markdown checklist." The model writes bash fluently. Let it figure out the curl flags, the jq path, the error cases. Run it, verify it works, adjust if needed.

2. **Consolidate via mise.** Put the result in `.mise/tasks/`. Add `set -euo pipefail`. Wire any secrets through `[env]` in `mise.toml` so they never appear in prompts. Run `mise run` to confirm it works on a clean shell. This is the moment the work gets captured: a tested, named command that lives outside any agent context.

3. **Write a skill that calls it.** A few lines of markdown. When the agent should invoke it, and what command to run. The implementation stays in the script. The skill just points to it.

The AI writes the complexity once. mise makes it repeatable. The skill makes it available without carrying it in context. That division of labor is the whole point.

The table is set. Bon appétit.

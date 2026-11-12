# Loggy for Claude Code

A Claude Code plugin for [Loggy](https://loggy-next.vercel.app), the logs and feature flags service for Next.js apps. With it, Claude can:

- **set up `loggy-next`** in your app: the package, the DSN, `<Loggy />` and `instrumentation.ts`;
- **debug from your real logs.** It finds the errors, failing requests and slow routes, follows one visitor's session across browser and server, and traces a problem to the deploy or feature flag value that brought it;
- **run your feature flags**, with a token that can make changes. It creates flags, switches them, writes their rules and percentage rollouts, and edits segments, environments, guards, alerts and saved views.

The plugin connects Claude Code to Loggy's MCP server, and adds a skill that teaches Claude how to use it well.

## Install

In Claude Code:

```
/plugin marketplace add Sekiraw/loggy-skill
/plugin install loggy@loggy
```

Claude Code then asks for a **Loggy API token**. Create one on the [AI assistants page](https://loggy-next.vercel.app/ai) of your dashboard:

- **Read only** (the default): Claude can read logs, flags, alerts and settings, and change nothing.
- **Can also make changes**: Claude can also do what a project member can. It changes only projects where your role allows it, and flag changes show in their history under your name. People, keys, billing and deleting projects stay in the dashboard either way.

The token is kept in your system's credential store, not in a settings file.

## Stay up to date

The plugin follows this repository's commits. Each one is published from Loggy's main repository, after a check that it matches the MCP server.

- To get updates on their own, turn on auto-update for the `loggy` marketplace under `/plugin` → Marketplaces.
- To update by hand, run `claude plugin update loggy@loggy`.

## Try it

- "Set up Loggy in this app."
- "What broke in production since the last deploy?"
- "Why did checkout fail for order ord_1234?"
- "Which routes got slower this week?"
- "Is the new-checkout flag bringing errors?"
- "Roll new-checkout out to 10% in production, and set a guard that turns it off if errors double."

## Without the plugin

You can add the MCP server yourself:

```bash
claude mcp add --transport http loggy https://loggy-next.vercel.app/api/mcp \
  --header "Authorization: Bearer loggy_pat_..."
```

If you do, copy `skills/loggy` to `~/.claude/skills/loggy` for the skill.

## Safety

- Anyone with a project's public key can send it logs, so a log can be written to look like an instruction. The skill and the server both tell Claude to treat log contents as data, and never to make a change because a log asks for it.
- Claude Code asks before each tool call unless you allow it, so you see every change before it happens.

## Contributing

This repository is published automatically from Loggy's main repository, which overwrites changes made here. Open an issue here for a bug or an idea.

`skills/loggy/references/tools.md` is generated from the MCP server.

## License

MIT

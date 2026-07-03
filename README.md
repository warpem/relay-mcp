# Relay Claude Code Plugin

A [Claude Code](https://claude.ai/code) plugin that connects Claude to your institution's Relay cryo-EM/cryo-ET workflow platform.

Installing this plugin gives Claude:
- **MCP tools** for reading and managing your Relay projects, spaces, jobs, and results
- **Domain knowledge** about the cryo-ET processing workflow and Relay's organization model

## Installation

**Step 1 — add this repo as a marketplace** (one-time):

```
/marketplace add https://github.com/warpem/relay-mcp
```

**Step 2 — install the plugin:**

```
/plugin install relay
```

Claude Code will prompt you for two values:

- **Relay server URL** — the base URL of your institution's Relay server (e.g. `https://relay.yourlab.edu`). Your cluster admin will provide this.
- **Personal access token** — generate one in Relay under your user settings. Tokens carry independent read/write/manage permissions per tier (projects, spaces, jobs), so ask your admin what access level you need.

That's it. The MCP server and domain skill load automatically on the next session.

### Updating

```
/plugin update relay
```

## What Claude can do

Once installed, Claude can help you:

- Browse your projects, spaces, and jobs
- Build and connect processing pipelines (import → motion correction → alignment → reconstruction → picking → refinement)
- Queue jobs on local or cluster compute
- Monitor running jobs and retrieve results
- Answer questions about cryo-ET workflow steps and Relay's job types

## Requirements

- Claude Code v2.1.154 or later
- A running Relay instance (self-hosted, managed by your institution)
- A personal access token from that instance
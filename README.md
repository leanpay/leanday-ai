# leanpay-ai — Claude Code AI infrastructure

> Leanpay's public marketplace for Claude Code AI infrastructure — skills, MCP servers, and extensions, shared openly with the community.

This repository is a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces). Install it once and the bundled plugins become available across all your projects, at the user level.

## Install

```
/plugin marketplace add leanpay/leanpay-ai
/plugin install leanpay@leanpay-ai
```

The `leanpay` plugin installs at the **user level** (`~/.claude`), so its skills work in every project. Pull future versions any time with:

```
/plugin marketplace update leanpay-ai
```

> **Note:** the repository is currently private. Public access (and the short `owner/repo` install address above working for everyone) is coming soon.

## What's inside

| Plugin | Description |
|---|---|
| [`leanpay`](./plugins/leanpay) | AI infrastructure skills and extensions — see the plugin README for details. |

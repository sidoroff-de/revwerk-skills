# RevWerk Skills

Public distribution for RevWerk's **Context Foundation Client Skills** — portable [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
that install into Claude Code, Cursor, Codex, Gemini CLI, and other SKILL.md-compatible agents. Each skill is a
thin client of a RevWerk-hosted Context Foundation MCP server: it carries no proprietary method of its own, and
is safe to publish and install anywhere.

Source for these skills is authored in the private `rev-ops-mcp` repo (`client-skills/`) and published here.
This repo is the public endpoint end users and clients connect to.

## Layout

```
skills/
  <skill-name>/
    SKILL.md          # source: frontmatter (name, description) + instructions
    references/        # optional supporting docs
    ...
dist/
  <skill-name>.skill   # packaged zip, one per skill, ready to install
.claude-plugin/
  plugin.json          # Claude Code plugin manifest (version-stamped on every publish)
  marketplace.json     # Claude Code marketplace manifest (lists the plugin(s) in this repo)
plugins/
  context-foundation/
    .codex-plugin/
      plugin.json      # Codex plugin manifest (version-stamped on every publish)
    skills/            # same 4 skills, duplicated here — Codex plugins can't source from repo root
.agents/
  plugins/
    marketplace.json   # Codex marketplace manifest (lists the plugin(s) in this repo)
.mcp.json              # the plugin's bundled MCP connector (URL only, no credentials)
VERSION                # canonical version stamp for this publish, mirrors plugin.json
```

`skills/` holds the human-readable source folders; `dist/` holds the packaged `.skill` zips built from that
source. Both are kept in sync and stamped with the same version, taken from the publishing repo's canonical
version file. `plugins/context-foundation/skills/` is a second copy of the same source, published for Codex —
see `client-skills/docs/adr/0002` in the private repo for why it's duplicated rather than shared with `skills/`.

## Install

Via the [skills.sh](https://skills.sh) installer (recommended):

```bash
npx skills@latest add sidoroff-de/revwerk-skills
```

Via the plugin/marketplace flow in Claude Code or Codex — the same two commands in both:

```
/plugin marketplace add sidoroff-de/revwerk-skills
/plugin install context-foundation@revwerk
```

In Claude Cowork and claude.ai, install through the UI instead: **Customize → Plugins → Add marketplace**, enter
`https://github.com/sidoroff-de/revwerk-skills`, then install the `context-foundation` plugin it lists.

## Connecting the MCP server

The plugin bundles its MCP connector (`.mcp.json`), so installing it also registers the hosted Context Foundation
server — there is no connector to add by hand.

That file carries a URL and nothing else. The server answers an unauthenticated request with a `401` and an OAuth
challenge, so your client registers itself and walks you through a sign-in that ends on the server's own `/license`
page. You paste your license key there, once. **The key is never stored in this repo, in the plugin, or in any
config file** — this repo is public and every client installs the identical copy.

If you need a static bearer token instead (scripts, CI, clients without OAuth support), configure the server
manually rather than relying on the bundled connector.

Manual / git:

```bash
git clone https://github.com/sidoroff-de/revwerk-skills ~/src/revwerk-skills
ln -s ~/src/revwerk-skills/skills/* ~/.claude/skills/
```

## Publishing

Content in `skills/`, `dist/`, `VERSION`, `.claude-plugin/plugin.json`, `plugins/context-foundation/skills/`,
and `plugins/context-foundation/.codex-plugin/plugin.json` is generated and synced from an internal publish
pipeline — don't hand-edit those paths; changes are made upstream and republished here.
Both marketplace manifests and `.mcp.json` are generated too. This README is the only file maintained directly
in this repo.

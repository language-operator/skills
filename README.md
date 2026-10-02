# language-operator skills

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) for the
language-operator org. It holds the skills the org's repos share, so each repo installs them
from here instead of keeping its own copy.

## What's here

| Path | What it is |
|---|---|
| `.claude-plugin/marketplace.json` | The marketplace, named `language-operator` |
| `plugins/langop/.claude-plugin/plugin.json` | The `langop` plugin |
| `plugins/langop/skills/<name>/SKILL.md` | One directory per skill |
| `plugins/langop/skills/<name>/scripts/*.sh` | Scripts a skill bundles |

The `langop` plugin has no skills yet. The first one, `iterate`, arrives with
[#2](https://github.com/language-operator/skills/issues/2).

## Install

### In a consuming repo

Commit a `.claude/settings.json` that declares the marketplace, pinned to a tag, and enables
the plugin:

```json
{
  "extraKnownMarketplaces": {
    "language-operator": {
      "source": {
        "source": "github",
        "repo": "language-operator/skills",
        "ref": "v0.1.0"
      }
    }
  },
  "enabledPlugins": {
    "langop@language-operator": true
  }
}
```

`v0.1.0` is the first tag and ships with #2. To take a newer release, change `ref`.

### From the shell

```bash
claude plugin marketplace add language-operator/skills
claude plugin install langop@language-operator
```

Then run `/plugin` inside Claude Code to see the marketplace and the plugin.

## Conventions

### Skill naming

- One directory per skill, `plugins/langop/skills/<name>/`, holding a `SKILL.md` with
  frontmatter.
- `<name>` is lowercase kebab-case and matches the frontmatter `name`. Name the skill for what
  it does (`iterate`, `prioritize`) and leave the plugin name out, since the plugin already
  namespaces it.
- Use the skills format. Don't add a legacy `commands/` directory.

### Per-repo detail lives in the consuming repo's `CLAUDE.md`

A skill here is the same text for every repo, so nothing repo-specific goes in it. When a
skill needs something that differs by repo, it reads a named section of that repo's
`CLAUDE.md`. For example, `/iterate` reads `## Testing` to learn how to test a change.

- Name the section the skill reads, by its exact heading, in the skill text.
- Say what the skill does when the section is missing. It should fall back to something
  sensible and tell the user it did.

### Scripts

Put a skill's scripts in `skills/<name>/scripts/` and call them through the plugin root, since
the plugin is installed outside the consuming repo:

```bash
bash "${CLAUDE_PLUGIN_ROOT}/skills/<name>/scripts/<script>.sh"
```

Scripts must pass `shellcheck`.

### Versioning

Releases are git tags named `vX.Y.Z`. The `version` in
`plugins/langop/.claude-plugin/plugin.json` matches the tag without the `v`, so bump it in the
commit that gets tagged. The bump is what delivers the release: Claude Code keeps an installed
plugin at its current version until that string changes. Consuming repos pin a tag with `ref`
and move to a new release by changing it.

## CI

`.github/workflows/lint.yaml` runs on pull requests and on pushes to `main`:

- `shellcheck` on `plugins/**/scripts/*.sh`
- `marketplace.json` and every `plugin.json` parse as JSON
- `claude plugin validate --strict` on the marketplace and on each plugin

To run the validation locally:

```bash
claude plugin validate --strict .
claude plugin validate --strict plugins/langop
```

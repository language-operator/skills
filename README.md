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

## Skills

| Skill | What it does | Reads from the repo's `CLAUDE.md` |
|---|---|---|
| [`iterate`](plugins/langop/skills/iterate/SKILL.md) | Works one GitHub issue from pick to merged PR to closed. Takes `[#issue] [--auto]`. | `## Testing` |

## Install

### In a consuming repo

Commit a `.claude/settings.json` that enables the plugin and declares the marketplace, pinned
to a tag:

```json
{
  "enabledPlugins": {
    "langop@language-operator": true
  },
  "extraKnownMarketplaces": {
    "language-operator": {
      "source": {
        "source": "github",
        "repo": "language-operator/skills",
        "ref": "v0.1.0"
      }
    }
  }
}
```

To take a newer release, change `ref`. The keys are in the order the `claude` CLI writes
them, so a later `claude plugin install` doesn't reorder the committed file.

What a teammate does after cloning depends on how they run Claude Code:

- **Interactive:** nothing. Claude Code adds the marketplace at the pinned `ref` and loads the
  plugin when a session starts in a trusted folder. On a fresh clone that is right after the
  "trust this folder" dialog; in a checkout that was already trusted it is the next start
  after the settings file arrives. There is no install prompt.
- **Non-interactive** (`claude -p`, scheduled or in-cluster agents): the settings file alone
  installs nothing, because there is no trust dialog to accept. Run these once, with the same
  tag the repo pins:
  ```bash
  claude plugin marketplace add 'language-operator/skills#v0.1.0'
  claude plugin install langop@language-operator --scope project
  ```

Two things to avoid in a repo that pins a tag:

- `claude plugin marketplace add language-operator/skills` without `#<tag>` follows `main`,
  and the install then takes `main`'s version, not the pinned one.
- `claude plugin marketplace add ... --scope project` rewrites the committed settings file
  and drops its `ref`.

### For yourself, in every repo

```bash
claude plugin marketplace add language-operator/skills
claude plugin install langop@language-operator
```

This follows `main`. Run `/plugin` inside Claude Code to see the marketplace and the plugin.

### Invoking a skill

A skill is registered as `/langop:<name>`, and the bare `/<name>` works too as long as
nothing else in the session has that name. So `/iterate` and `/langop:iterate` both run the
shared skill.

A repo that still has its own `.claude/commands/iterate.md` shadows the bare name: `/iterate`
runs the local copy and only `/langop:iterate` reaches the plugin. Delete the local copy when
the repo adopts the plugin.

## Verified behaviour

The Claude Code docs were ambiguous on four points, so each was tested, on Claude Code
2.1.287. The details are in [Install](#install).

| Question | Answer |
|---|---|
| Does the skill resolve as `/iterate` or only as `/langop:iterate`? | Both. The bare name works unless something else in the session has it, such as a leftover `.claude/commands/iterate.md`. |
| Is `enabledPlugins` an object or an array? | An object: `{"langop@language-operator": true}`. The full working file is under [In a consuming repo](#in-a-consuming-repo). |
| Is a teammate who clones a repo with that file prompted to install? | No, and they don't need to be. Interactive sessions load the plugin once the folder is trusted. Non-interactive runs need the two commands above, once. |
| Does `claude plugin validate` exist? | Yes. CI runs it with `--strict` on the marketplace and on each plugin. It needs no credentials. |

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

Allow each script in the skill's `allowed-tools`, written with the same quotes as the call:

```yaml
allowed-tools: Bash(bash "${CLAUDE_PLUGIN_ROOT}/skills/<name>/scripts/<script>.sh" *)
```

The quotes have to match. A rule without them does not cover the quoted call, the script is
refused with "This command requires approval", and an unattended run has nobody to approve
it.

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

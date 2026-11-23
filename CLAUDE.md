# rainmore-marketplace

A Claude Code plugin marketplace (`.claude-plugin/marketplace.json`) whose plugins live in
`plugins/<plugin>/`. Each skill is `plugins/<plugin>/skills/<skill>/SKILL.md`. The repository is
`rainmore/claude-marketplace` on GitHub. The marketplace name is `rainmore-marketplace`, because names
starting with `claude-` are rejected as impersonating official marketplaces.

## Versioning convention

- **One version for everything.** Every plugin's `plugin.json` `version`, every plugin entry's
  `version` in `marketplace.json`, and the marketplace's `metadata.version` always hold the same
  value.
- **Bump all of them together** on every release, including plugins that did not change. Never bump
  one plugin on its own.
- **Semver:** patch for wording fixes, minor for a new or changed skill, major when a rule is
  reversed or removed.
- **Bump only for a release.** Edits between releases don't change the version. Bump once, in the
  release commit.
- **One tag per release, for the whole repo:** the bare version, e.g. `0.1.0` (annotated, no `v` prefix). No per-plugin tags: don't
  use `claude plugin tag`, which creates `<plugin>--v<version>`.
- **Before tagging, confirm the versions agree and the manifests validate:**
  ```bash
  grep -rhoE '"version": *"[^"]+"' .claude-plugin plugins/*/.claude-plugin | sort -u   # one line
  claude plugin validate .
  ```

The full release steps are in README.md under "Releasing a new version".

## Other rules

- No Python, and no hooks or scripts, in the plugins. They contain skills (Markdown) and manifests only.
- When adding a plugin, add it to `marketplace.json` at the current shared version, and to the
  README tables.
- Commit and tag locally. Pushing needs the user's GitHub credentials, so the user pushes.

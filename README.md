# claude-marketplace

A Claude Code plugin marketplace.

## Plugins

| Plugin | Description |
| --- | --- |
| [java-spring](plugins/java-spring) | Skills for Java and Spring development |

### java-spring skills

| Skill | Purpose |
| --- | --- |
| `java-code-placement` | No new utility classes or public constants: where the logic or value belongs instead (owning type, bean, enum, record, properties) |
| `java-spring-conventions` | Java / Spring Boot coding conventions: injection, Lombok, validation, logging, exceptions, `var`, MapStruct, records vs POJOs, Temporal |
| `java-test-generator` | Generate unit tests for a class: Mockito, AssertJ, parameterized tests |
| `java-unit-tests` | JUnit 5 + Mockito + AssertJ conventions; self-describing test names, no `@DisplayName` |
| `mapstruct-generator` | Generate MapStruct mappers with named qualifiers, custom mappings, and null handling |
| `spring-data-querydsl` | Repositories on a QueryDSL base, `*Specification` conditions, soft delete, native SQL in `*Sql` classes, JSON columns |
| `db-job-queue` | PostgreSQL job queues: `SKIP LOCKED` claims, stale recovery, release limits, workers and pollers, transaction pitfalls |
| `spring-integration-tests` | `integrationTest` source set, Testcontainers PostgreSQL, testing real SQL and migrations, standalone MockMvc, `Clock.fixed` |
| `spring-boot-configuration` | `@ConfigurationProperties` as records, record vs POJO guide, env-var defaults, injected `Clock`, rerunnable SQL migrations |

## Install

Each step works as a slash command inside Claude Code or as a `claude` CLI command in your shell.

### 1. Add the marketplace

```bash
/plugin marketplace add rainmore/claude-marketplace                          # GitHub
/plugin marketplace add https://github.com/rainmore/claude-marketplace.git  # git URL
/plugin marketplace add /path/to/claude-marketplace                          # local clone
```

Shell equivalent: `claude plugin marketplace add rainmore/claude-marketplace`.

The repository is `claude-marketplace`, but the marketplace registers as **`rainmore-marketplace`**
(the `name` in `.claude-plugin/marketplace.json`; names starting with `claude-` are reserved for
official marketplaces). The commands below use that name.

### 2. Install the plugin

```bash
/plugin install java-spring@rainmore-marketplace
```

Shell equivalent: `claude plugin install java-spring@rainmore-marketplace --scope user`.

| Scope | Applies to | Use when |
| --- | --- | --- |
| `user` (default) | you, in every project | you want the skills everywhere |
| `project` | everyone working in this repo (written to `.claude/settings.json`) | the team should share the plugin |
| `local` | you, in this repo only (not committed) | trying it out in one project |

### 3. Check it

```bash
/plugin                     # Installed tab shows java-spring and its version
claude plugin list          # same, from the shell
```

Skills from the plugin are namespaced, e.g. `java-spring:java-unit-tests`. Claude loads them on its
own when a task matches a skill's description, and you can also invoke one by name.

If you previously copied any of these skills into `~/.claude/skills/` or a project's
`.claude/skills/`, delete those copies. Otherwise Claude sees two versions of the same skill.

### Team setup

To have everyone working in a repo pick up the marketplace, commit this to that repo's
`.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "rainmore-marketplace": {
      "source": { "source": "github", "repo": "rainmore/claude-marketplace" },
      "autoUpdate": true
    }
  },
  "enabledPlugins": {
    "java-spring@rainmore-marketplace": true
  }
}
```

Each member must accept the folder's workspace trust prompt before the marketplace is registered,
and then install the plugin once:

```bash
claude plugin install java-spring@rainmore-marketplace --scope project
```

## Keeping the plugin up to date

Claude Code decides whether a newer version exists from the `version` in
`plugins/java-spring/.claude-plugin/plugin.json`. A change only reaches users once that version is
bumped (see [Releasing](#releasing-a-new-version)).

### Automatically

Turn on auto-update for this marketplace. It is **off by default** for third-party marketplaces.

- **In Claude Code:** run `/plugin`, open **Marketplaces**, select `rainmore-marketplace`, and enable
  auto-update.
- **For a team:** set `"autoUpdate": true` on the marketplace entry, as in [Team setup](#team-setup).

When an update is installed, Claude Code shows a notice. Run `/reload-plugins` or start a new
session to use it.

### Manually

```bash
claude plugin marketplace update rainmore-marketplace   # fetch the latest catalog
claude plugin update java-spring@rainmore-marketplace   # install the newer version
```

Then run `/reload-plugins` or restart Claude Code. `claude plugin list` shows which version is
installed.

### Staying on a fixed release

To stay on a tagged release instead of following `main`, add the marketplace at that tag:

```bash
/plugin marketplace add rainmore/claude-marketplace#0.1.0
```

To move to a later release, remove the marketplace (`claude plugin marketplace remove
rainmore-marketplace`) and add it again at the new tag.

### Uninstall

```bash
claude plugin uninstall java-spring@rainmore-marketplace
claude plugin marketplace remove rainmore-marketplace
```

## Releasing a new version

Every plugin in this marketplace shares one version, which is also the marketplace's
`metadata.version`. A release bumps all of them together, even plugins that did not change, and is
tagged once for the whole repository with the bare version, e.g. `0.1.0` (no `v` prefix). There are no per-plugin tags.

1. Make the skill changes.
2. Bump `version` to the same new value in **every** place:
   - `metadata.version` in `.claude-plugin/marketplace.json`
   - each plugin entry in `.claude-plugin/marketplace.json`
   - each `plugins/<plugin>/.claude-plugin/plugin.json`

   Use a patch bump for wording fixes, a minor bump for a new or changed skill, and a major bump
   when a rule is reversed.
3. Check that the versions match, and validate:
   ```bash
   grep -rhoE '"version": *"[^"]+"' .claude-plugin plugins/*/.claude-plugin | sort -u   # must print one line
   claude plugin validate .
   ```
4. Commit, then tag: `git tag -a <version> -m "Release <version>"`, e.g. `git tag -a 0.1.0 -m "Release 0.1.0"`.
5. Push the commit and the tag: `git push --follow-tags`.

Users with auto-update get the new version automatically. Everyone else runs the
[manual update](#manually).

## Layout

```
.claude-plugin/marketplace.json        # marketplace catalog
plugins/
  java-spring/
    .claude-plugin/plugin.json         # plugin manifest
    skills/
      <skill-name>/SKILL.md            # one directory per skill
```

## Adding a skill to java-spring

1. Create `plugins/java-spring/skills/<skill-name>/SKILL.md` with `name` and `description`
   frontmatter. The description decides when Claude loads the skill, so state what it covers, when
   to use it, and its one rule that must not be broken.
2. Add the skill to the table at the top of this README.
3. Release it as described in [Releasing a new version](#releasing-a-new-version).

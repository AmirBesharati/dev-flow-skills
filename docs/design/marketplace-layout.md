# Marketplace layout

Status: proposal. Nothing below exists yet except this doc and the skill template in `templates/skill/`.

## Goal

One public repo that a team adds to Claude Code once, so every developer on the team gets the same dev workflows (branching, planning, TDD, review, release) as installable skills and commands. Teams pick the bundles they need and can pin a version.

## What we learned from the references

**mattpocock/skills** ships one plugin from one repo:

- Skills live in bucket folders: `skills/engineering/`, `skills/productivity/`, `skills/misc/`, `skills/in-progress/`, `skills/deprecated/`. Only the promoted buckets are listed in `.claude-plugin/plugin.json`, so the bucket name decides what ships.
- Each skill is a folder with `SKILL.md` (frontmatter `name`, `description`, optional `disable-model-invocation`) plus supporting reference files beside it, and an `agents/openai.yaml` so the same skill works in Codex.
- Every skill is either user-invoked (a slash command for humans) or model-invoked (Claude may reach for it on its own). Skills call each other through the Skill tool by name, never by `../other/FILE.md` links.
- A router skill (`ask-matt`) maps every skill and how they chain. It must be updated whenever a skill is added or renamed.
- Versioning is Changesets: one `.changeset/*.md` per change, a release workflow bumps `package.json`, and a script copies the version into `plugin.json`.
- `SCOPE.md` and `.out-of-scope/` record what gets accepted and what was rejected, which keeps contributions on track.

**Claude Code plugin marketplaces**:

- A marketplace is a repo with `.claude-plugin/marketplace.json` listing plugins (`name`, `source`, `description`, `version`, `category`, `keywords`). `source` can be a relative folder in the same repo or another git repo.
- A plugin is a folder with `.claude-plugin/plugin.json` and any of `skills/`, `commands/`, `agents/`, `hooks/hooks.json`, `.mcp.json`.
- Users run `/plugin marketplace add owner/repo` and `/plugin install <plugin>@<marketplace>`. A team commits `extraKnownMarketplaces` and `enabledPlugins` to the project's `.claude/settings.json` so everyone who trusts the folder gets the same set.

## The main decision: many small plugins, not one big one

Matt's repo is one person's curated set, so one plugin fits. A team marketplace needs choice: a backend team does not want frontend skills, and a team may adopt the git flow before the review flow. So this repo ships **one marketplace with several plugins**, one per workflow area, plus a `core` plugin every team installs.

## Repository layout

```
dev-flow-skills/
├── .claude-plugin/
│   └── marketplace.json          # lists every plugin below
├── plugins/
│   ├── core/                     # always installed: router + shared conventions
│   │   ├── .claude-plugin/plugin.json
│   │   ├── skills/
│   │   │   ├── dev-flow/         # router: which skill for which step
│   │   │   └── setup-dev-flow/   # one-time per-repo setup (tracker, labels, branch names)
│   │   └── README.md
│   ├── planning/                 # spec, tickets, grilling a plan
│   ├── git-flow/                 # branch naming, commits, PRs, release notes
│   ├── testing/                  # TDD loop, writing tests, fixing flaky tests
│   ├── code-review/              # self-review, reviewing others, review checklist
│   ├── debugging/                # reproduce, bisect, root-cause
│   ├── docs/                     # ADRs, READMEs, changelogs
│   └── stack-<name>/             # optional stack packs, e.g. stack-react, stack-django
├── templates/
│   ├── skill/                    # copy this to start a new skill
│   └── plugin/                   # copy this to start a new plugin
├── docs/
│   ├── design/                   # design docs like this one
│   ├── adr/                      # decisions, numbered
│   └── <plugin>/<skill>.md       # human-facing page per promoted skill
├── scripts/
│   ├── validate.mjs              # checks manifests, frontmatter, README links
│   └── sync-versions.mjs         # copies changeset versions into plugin.json files
├── .changeset/
├── .github/workflows/            # validate on PR, release on main
├── CONTRIBUTING.md
├── SCOPE.md                      # what belongs here, and what does not
├── CLAUDE.md                     # rules for agents editing this repo
└── README.md                     # install block + catalogue
```

Each plugin folder looks the same:

```
plugins/<plugin>/
├── .claude-plugin/plugin.json    # name, version, description, skills list
├── skills/<skill-name>/
│   ├── SKILL.md
│   ├── agents/openai.yaml        # Codex metadata, optional but encouraged
│   └── <REFERENCE>.md            # long details the skill loads only when needed
├── commands/                     # only for thin slash commands that wrap a skill
├── hooks/hooks.json              # only when the workflow needs enforcement (e.g. block force-push)
└── README.md                     # every skill with a one-line description
```

## Naming

- Plugins: kebab-case nouns for the workflow area (`git-flow`, `code-review`). Stack packs start with `stack-`.
- Skills: kebab-case, verb-first when user-invoked (`open-pr`, `write-spec`), noun or gerund when model-invoked (`tdd`, `debugging`). Names are unique across the whole marketplace, not only within a plugin, so the router and cross-skill calls stay unambiguous.
- The marketplace is named `dev-flow`, so installs read `/plugin install git-flow@dev-flow`.

## Categories and maturity

Categories are the plugins themselves, and the `category` field in `marketplace.json` mirrors them. Maturity is separate from category, so it is a field, not a folder:

- `stable`: listed in the plugin's `plugin.json` and shipped.
- `beta`: lives in the plugin folder, listed in its README under "Beta", not in `plugin.json`. Users try it by copying or symlinking.
- `deprecated`: moved to `plugins/<plugin>/deprecated/`, kept for one minor release, then removed.

`scripts/validate.mjs` enforces that only `stable` skills appear in `plugin.json`.

## Skill template

`templates/skill/SKILL.md` holds the starting point. Its shape:

```markdown
---
name: <skill-name>
description: <one line. User-invoked: what it does for a human. Model-invoked: "Use when ..." trigger phrasing.>
# disable-model-invocation: true   # uncomment for user-invoked skills
---

# <Title>

<One paragraph: what this skill produces and when it stops.>

## When to use
## Before you start   (inputs, repo config it reads, skills it calls first)
## Steps            (numbered, each one observable)
## Done when        (a checklist the agent verifies before finishing)
## Related skills   ("Call the Skill tool with <name>" for each dependency)
```

Rules every skill follows:

- Keep `SKILL.md` short; put long reference material in sibling files the skill reads on demand.
- Depend on other skills by calling the Skill tool with their name, never by linking into another skill's folder.
- Read team-specific values (tracker, branch prefix, label names) from the config that `setup-dev-flow` writes into the target repo, never hard-code them. This is what lets one marketplace serve many teams.
- Each skill also gets `docs/<plugin>/<skill>.md` with four sections: What it does, When to reach for it, Common questions, It's working if.

## Versioning

- Each plugin has its own semver in `plugin.json`. A breaking change to a skill (renamed, removed, or a changed contract with the per-repo config) is a major bump of that plugin.
- Changesets drive releases: every PR that changes a skill adds a `.changeset/*.md` naming the plugin and the bump. The release workflow opens a version PR, and `scripts/sync-versions.mjs` writes the new versions into each `plugin.json` and the `marketplace.json` entries.
- Each release is tagged `<plugin>@<version>` (e.g. `git-flow@1.2.0`) so teams can pin.
- `CHANGELOG.md` lives per plugin.

## How a team installs it

One developer tries it:

```
/plugin marketplace add AmirBesharati/dev-flow-skills
/plugin install core@dev-flow
/plugin install git-flow@dev-flow
```

A team standardises on it by committing this to their project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "dev-flow": {
      "source": { "source": "github", "repo": "AmirBesharati/dev-flow-skills" }
    }
  },
  "enabledPlugins": {
    "core@dev-flow": true,
    "git-flow@dev-flow": true,
    "code-review@dev-flow": true
  }
}
```

Then someone runs `/setup-dev-flow` once in the repo; it writes the team's choices (tracker, branch naming, label vocabulary) to `docs/agents/dev-flow.md` and adds a pointer in `CLAUDE.md`. To pin a version, the team points the marketplace source at a tag (`"ref": "git-flow@1.2.0"`) or forks the repo.

A team that wants to change a skill forks the repo and points `extraKnownMarketplaces` at the fork. Fixes that help everyone come back upstream as PRs.

## Quality gates

- `claude plugin validate . --strict` and `scripts/validate.mjs` run on every PR.
- Validation checks: every stable skill is in `plugin.json`, in its plugin README, and in the router; frontmatter has `name` matching the folder and a `description`; no cross-skill file links; a changeset exists when a skill changed.
- `SCOPE.md` sets the bar for new skills: it must describe a workflow a real team ran and wants repeated, and must not be composable from existing skills.

## Open questions

1. Should the first release ship only `core`, `git-flow` and `code-review`, and add the rest later? Recommended: yes, small and stable beats broad.
2. Should skills also carry `agents/openai.yaml` for Codex from day one? Recommended: yes, it is cheap and widens adoption.
3. Do stack packs live here or in separate repos referenced from `marketplace.json`? Recommended: here until one grows its own maintainers.

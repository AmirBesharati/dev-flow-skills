---
name: skill-name
description: One line. For a user-invoked skill, say what it does for a person browsing slash commands. For a model-invoked skill, use trigger phrasing such as "Use when the user wants to ...".
# disable-model-invocation: true
---

# Skill Title

One paragraph: what this skill produces, and when it stops.

## When to use

- The situations this skill is for.
- The situations it is not for, and which skill to use instead.

## Before you start

- Inputs the skill needs from the user.
- Repo config it reads (written by `setup-dev-flow`), such as tracker, branch prefix and label names.
- Skills to run first. Call the Skill tool with their name.

## Steps

1. Each step is something the agent does and can check.
2. Keep long reference material in a sibling file and read it only when the step needs it.

## Done when

- [ ] The output exists where the team expects it.
- [ ] Anything the skill changed is verified.

## Related skills

- Call the Skill tool with "other-skill" when ...

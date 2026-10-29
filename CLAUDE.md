# Working in albilities

A Claude Code plugin with skills for the loop around a change, from planning
a branch to getting its PR ready to merge.

## Layout

- `skills/<name>/SKILL.md`: one directory per skill.
- `fragments/<group>/<name>.md`: shared content, used by two or more skills.
- `bin/`: scripts that skills and hooks call.
- `hooks/hooks.json`: plugin hooks.
- `rules/`: always-loaded text meant for the user's own CLAUDE.md. It is never
  installed automatically.

## Writing skills

- **Frontmatter.** `name` and `description`. The description puts the key use
  case first and states triggers and anti-triggers. User-only skills set
  `disable-model-invocation: true`.
- **Mandatory fragments are injected through `bin/include-fragment`,** each on
  its own line, and the skill pre-approves the script in `allowed-tools`:

  ```
  allowed-tools: Bash(${CLAUDE_PLUGIN_ROOT}/bin/include-fragment *)
  ```
  ```
  !`${CLAUDE_PLUGIN_ROOT}/bin/include-fragment bars/authoring`
  ```

  Never use `cat`: Claude Code blocks it for files outside the working
  directory, even when pre-approved. A missing fragment aborts the skill.
- **Conditional fragments are run, not linked:** tell the model to run
  `${CLAUDE_PLUGIN_ROOT}/bin/include-fragment go/design` when it needs them.
  Reading a plugin file directly prompts outside plan mode.
- **`bin/include-fragment` replaces `${CLAUDE_PLUGIN_ROOT}` inside
  fragments,** so a fragment can name a script by its full path. It takes
  several names, so related fragments load in one call.
- **Skills that call `bin/backlog`** pre-approve
  `Bash(${CLAUDE_PLUGIN_ROOT}/bin/backlog *)`.
- **Pre-approved commands must be called plainly:** one command per Bash call,
  with no shell variables, pipes, `&&` or loops. Compound commands don't match
  `allowed-tools` patterns and prompt. Skills say this wherever they rely on a
  pre-approval.
- **Each Bash call is a fresh shell.** Never set a variable in one command and
  read it in another. Resolve a value once and write it literally into later
  commands. Temp files go under one `mktemp -d` directory whose literal path is
  reused, never into the user's repo.
- **Free text** (bodies, titles, comments) goes through a file or single
  quotes, never double quotes, because backticks run under zsh.
- **Run `bin/check-plugin` after any change.** It catches missing fragments,
  missing pre-approvals and em dashes.
- **Gates go at the top of each body.** After compaction only the first 5k
  tokens of a skill are kept.
- **Keep SKILL.md short.** Aim for under 200 lines; move shared substance into
  fragments.
- **A fragment is self-contained prose.** It has no frontmatter and doesn't
  name the skills that use it.

## Writing style

- Every file follows `fragments/bars/authoring.md`: answer first, plain
  sentences, no em dashes, no filler, cut what the reader already has.

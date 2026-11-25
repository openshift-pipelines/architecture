---
id: ADR-0009
title: Offline command search in tkn (tkn search)
date: 2026-11-25
status: proposed
authors:
  - "@divyansh42"
reviewers: []
tags:
  - cli
  - tkn
  - search
---
<!-- Status lifecycle:
- Proposed: Under discussion, not yet accepted
- Accepted: Approved, ready for implementation
- Implemented: Done and reflected in the product
- Deprecated: Not required anymore
- Superseded: Replaced by a newer ADR (link to it)
- Rejected: Not accepted (stays in repo for reference)
-->
---

# ADR-0009: Offline command search in tkn (`tkn search`)

## Context

`Required`

Users and AI agents often know *what* they want ("show logs of the failed run") but not *which* `tkn` command does it.

- Existing problem: there is no way to go from a description of the task to the matching `tkn` command.
- Constraints: it must work offline, with no cluster, credentials or network.
- Use cases: people looking for the right command, and AI agents that need machine-readable results.
- Impacted: the `tkn` CLI.

## Decision

`Required`

Add an experimental command, `tkn search "<what you want to do>"`, that suggests the right `tkn` command.
It searches tkn's own help text using standard keyword ranking (BM25, used by most search engines) with synonyms
and typo tolerance. It returns a ranked list of commands, each with its description, an example, its matching
flags and a score. No AI / LLM.

- Chosen option: offline keyword search over tkn's own help text, over an online AI service (the Azure `az find` approach).

### Architectural Overview

`Required`

The search data is built on every run from tkn's own commands and help text. It always matches the installed
tkn and there is nothing to generate or keep in sync. With ~75 commands, it's instant.

1. Make a list of every tkn command that does something (e.g. `tkn pipelinerun logs`, not the group
   `tkn pipelinerun`), using its name, aliases, description and flags from tkn's built-in help.
   Installed plugins (`tkn-*` programs on PATH) are added by name only, because tkn can't read their help.
2. Clean up the query: lowercase, drop filler words (`the`, `of`, …), reduce words to a base form
   (`logs` → `log`, `failed` → `fail`).
3. Expand the query to catch other wordings (see below).
4. Score every command, show the best 5 (`--limit` changes this) with up to 3 matching flags each.

**Where a match counts most**

| Part of the command | Weight |
| --- | --- |
| Name and aliases (`pipelinerun`, `pr`, `logs`) | highest |
| One-line description, plus optional extra keywords added by tkn developers | high |
| Flag names and descriptions | medium |
| Long description and examples | low |

**Other wordings that still match** (exact words always count most)

| Kind | Example |
| --- | --- |
| Synonyms | `stop` → cancel · `run` → start · `remove` → delete · `latest` → last · `preview` → dry-run |
| Words typed separately | `pipeline run` → pipelinerun |
| Partial words | `pipe` → pipeline |
| Typos | `cancle` → cancel |

**Ranking rules**

- Words found in almost every command count less. `namespace` barely helps, because nearly every command has
  `--namespace`. `dry-run` is a strong hint, because only a few commands have it.
- A command does not rank higher just because its help text is long and mentions many words. A match in a short
  description counts more than the same word buried in a long one.
- Deprecated commands and plugins rank lower.
- Weak matches are dropped. The best result always shows score 1.00, and the others are scored relative to it.

**Example:** for "show logs of the last failed pipelinerun", `tkn pipelinerun logs` ranks first because
"pipelinerun" and "logs" are in its name and `--last` / `--log-failed` match its flags.
`tkn taskrun logs` comes second because it has no "pipelinerun".

### Implementation Details

`Required`

- Command: `tkn search "<what you want to do>"`, marked experimental.
- Flags: `--limit` (default 5); `-o json` / `-o yaml`. The default is a table, as with other tkn commands.
- Results are commands plus their matching flags, because many tasks are really flags (`--last`, `--dry-run`, `--keep`).
- No match: a friendly message (an empty list in JSON), exit code 0.
- Empty query, or an invalid `-o` / `--limit` value: an error, exit code 3 (invalid input).
- Affected components: `tkn` CLI (new `search` command, `tkn --help`, unknown-command errors).

*Scores below are examples; real values come from tuning.*

**For humans (default table)**

```console
$ tkn search "show logs of the last failed pipelinerun"
1. tkn pipelinerun logs  (score 1.00)
   Show the logs of a PipelineRun
   Flags:    --log-failed  show logs for failed tasks and steps only
             -L, --last    show logs for last PipelineRun
   Example:  tkn pipelinerun logs foo -n bar

2. tkn taskrun logs  (score 0.57)
   Show TaskRuns logs
   Flags:    -L, --last    show logs for last TaskRun
   Example:  tkn taskrun logs foo -n bar
```

**For AI agents (`-o json`; `-o yaml` also works)**

```json
{
  "query": "show logs of the last failed pipelinerun",
  "results": [
    {
      "command": "tkn pipelinerun logs",
      "aliases": ["tkn pr logs"],
      "type": "command",
      "description": "Show the logs of a PipelineRun",
      "usage": "tkn pipelinerun logs",
      "example": "tkn pipelinerun logs foo -n bar",
      "flags": [
        {"name": "--log-failed", "description": "show logs for failed tasks and steps only"},
        {"name": "--last", "shorthand": "-L", "description": "show logs for last PipelineRun"}
      ],
      "score": 1.0
    }
  ]
}
```

**More examples**

| You type | Top result |
| --- | --- |
| `start a pipeline with parameters` | `tkn pipeline start` (flag `--param`) |
| `stop a running pipeline run` | `tkn pipelinerun cancel` |
| `delete old pipelineruns but keep the latest 5` | `tkn pipelinerun delete` (flag `--keep`) |
| `preview a pipelinerun without running it` | `tkn pipeline start` (flag `--dry-run`) |
| `cancle pipelnerun` (typos) | `tkn pipelinerun cancel` |
| `results` (with the `tkn-results` plugin installed) | `tkn results` (plugin) |

**Pointing people to it**

- `tkn --help` gets one new line: `Not sure which command to use? Describe what you want to do: tkn search "..."`
- Unknown-command errors get one extra line at the end:

```console
$ tkn pr showlog
Error: unknown command "showlog" for "tkn pipelinerun"

Did you mean this?
	logs

To find a command by what you want to do, run: tkn search "showlog"
```

## Security & Compliance

`Optional`

- No cluster, credentials or network access. The command reads only tkn's own help text and the names of
  `tkn-*` plugins on PATH.

## Compatibility & Migration

`Optional`

- New, experimental command. Existing output changes only by one new line in `tkn --help` and one extra line
  at the end of unknown-command errors.
- Ranking and output may still change while the command is experimental.

## Testability

`Required`

- A test with ~30 real queries catches ranking regressions. Poor results show which help text needs improving.

## Scope

`Required`

- **In-Scope:**
  - `tkn search` command with table, JSON and YAML output
  - Matching on command names, aliases, descriptions, flags and examples, including synonyms, partial words and typos
  - Installed plugins, by name
  - Hints in `tkn --help` and unknown-command errors
- **Out-of-Scope:**
  - AI / LLM search
  - Anything that needs a cluster, credentials or network
  - Plugin help text (tkn can't read it)

## Acceptance Criteria

`Required`

The decision is considered accepted when:

- [ ] Problem, scope, and non-goals are clear
- [ ] Alternatives were considered and compared
- [ ] Chosen option's trade-offs (not just benefits) are documented
- [ ] Backward compatibility / migration path addressed
- [ ] Security/RBAC impact assessed
- [ ] Required reviewers approved
- [ ] Open concerns resolved or tracked
- [ ] Decision is feasible to implement

## Decision Drivers

`Required`

- Works offline: no cluster, credentials or network needed.
- Always matches the installed tkn, with nothing to generate or keep in sync.
- Predictable results, no new dependencies, no AI.
- Serves both people (table) and AI agents (`-o json`), with the same output options as other tkn commands.

## Consequences

`Required`

- Positive outcomes:
  - Users and AI agents find the right command from a description of what they want to do.
  - Results include the matching flags.
  - Agents learn which plugins are installed.
  - `tkn --help` and unknown-command errors point people to the search.
- Risks:
  - Results depend on help-text wording. The ~30-query test catches regressions, and poor results show which
    help text needs improving.
  - Typo matching could give odd results. Typo and partial-word matches always count less than exact words.
  - Output may change. The command stays experimental until feedback settles it.
- Follow-up:
  - Remove the experimental status once feedback settles ranking and output.

## Dependencies

`Required`

- No new dependencies. The command needs no cluster, so there is no Tekton or OpenShift version requirement.

## References

`Required`

- [JIRA: SRVKP-15078](https://redhat.atlassian.net/browse/SRVKP-15078)
- [CLI improvement proposal, §4.2](https://gist.github.com/chmouel/27e7a094e40becf24f9a067279c0e0a7)

## Change Log

`Required`

| Date | Author | Change Summary |
| --- | --- | --- |
| 2026-11-25 | @divyansh42 | Created initial ADR |

---
id: ADR-0007
title: Release notes from upstream pull requests to Jira
date: 2026-10-19
status: proposed
authors:
  - "@psrvere"
reviewers:
  - "Navya Tatineni"
  - "Emil Natan (@enarha)"
  - "Jawed Khelil (@jkhelil)"
  - "Akshay Pant (@theakshaypant)"
  - "Divyanshu Agrawal (@divyansh42)"
  - "Kavya Verma"
tags:
  - release
  - release-notes
  - jira
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

# ADR-0007: Release notes from upstream pull requests to Jira

## Context

`Required`

The docs team writes the OpenShift Pipelines release notes from one [Jira dashboard](https://redhat.atlassian.net/jira/dashboards/22264), the docs dashboard. A change shows up there only as an SRVKP ticket with the release's fix version, in status Dev Complete, Release Pending or Closed, and with four fields set.

| Field | Who sets it |
|---|---|
| Release Note Type | Engineering |
| Release Note Text | Engineering |
| Release Note Status | Engineering sets Proposed. The docs team sets Done after QE review and the docs merge. This ADR adds a step before Proposed, see the decision. The field tracks the note only. The ticket's own status is separate, and this ADR does not change how it moves |
| Git Pull Request | Engineering |

Today an engineer writes the note in the upstream pull request, then copies it into the ticket by hand. Nothing reminds them, and nothing checks the ticket until the docs deadline. So the docs team often starts with far fewer notes than the release has changes. To fill the gap, someone collects the upstream notes for each component near the end of the release and hands them to the docs team. Nobody checks those notes against what shipped downstream; a writer reviews them and adds them to the docs. This ADR replaces that second path: every note reaches the docs team through a ticket.

This ADR covers minor releases, such as 1.25.0. Patch releases are out of scope. Their notes list only bug fixes, and Akshay Pant's audit tool already lists the tickets of a patch release that have no note, as in the [1.24.2 audit](https://github.com/theakshaypant/osp-rc-tools/blob/jira/config/1.24.2/audit-release-1.24.2.md#p12n-console-plugin). No dashboard is needed for them.

Claude produced the numbers below from the GitHub and Jira APIs. For each component, it compared the upstream branch 1.23 was built from with the one 1.24 was built from, as hack's release files for [1.23](https://github.com/openshift-pipelines/hack/blob/7b881a425ecfc88158d3f49fbfb07b5c3f327df8/config/downstream/releases/1.23.yaml) and [1.24](https://github.com/openshift-pipelines/hack/blob/7b881a425ecfc88158d3f49fbfb07b5c3f327df8/config/downstream/releases/1.24.yaml) list them, and took the pull requests behind the new commits. It rebuilt each 1.24.0 ticket from its change history, as it stood at the release-note freeze on 2026-08-28 and in the weeks after. Where a published entry did not name its pull request, Claude matched it from the text. The appendices list every match so readers can check them.

- 477 pull requests went into the release. They were merged into 10 upstream repositories between the branches 1.23 and 1.24 were built from, as hack's release files list them. Pull requests opened by bots are left out. [Appendix A](#appendix-a-the-1240-range-per-repository) lists the repositories, their branches and the components left out.
- 155 of them needed a release note. The count comes from each pull request's text, not from reading its code. Claude counted a pull request as needing a note when its author wrote a release note in it, 96 pull requests, or when it had no release-note block and its title started with feat, fix, perf or security, 59 pull requests. The 59 are a guess from the title, and some of them need no note.
- 110 of those 155 had no SRVKP ticket yet, so they could not reach the docs dashboard.
- 3 of the 155 had every field set.

### What happened after the freeze

Between the freeze and publication, a handful of tickets were edited, and 31 were marked Release Note Not Required in one pass, which cleared them from the docs dashboard's "needs attention" list. None of this added a note to the 155 changes, because nobody had a list of the gaps to work through. When the docs team published the 1.24.0 notes on 2026-09-17, 7 of the 155 changes had every field set. No step in the process asks anyone to go back and create the missing tickets.

### Working backwards from the published notes

The [1.24.0 release notes](https://docs.redhat.com/en/documentation/red_hat_openshift_pipelines/1.24/html-single/release_notes/index) have 30 entries. Claude traced each one back to the upstream pull request behind it. The full list, with every pull request and ticket, is in [Appendix B](#appendix-b-1240-entries-and-their-pull-requests).

| What the entry traces to | Entries | How it reached the docs team |
|---|---|---|
| A pull request in the 1.24.0 range, with its own SRVKP ticket | 12 | The normal path. The ticket carried the note |
| A pull request in the range, with no ticket of its own | 12 | The owner listed them by hand in the component's "Upstream release notes" story. The 1.24 epic "Upstream release notes for 1.24", [SRVKP-14261](https://redhat.atlassian.net/browse/SRVKP-14261), has one such story per component. 11 entries came from "Upstream release notes for PAC", [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264), and 1 from "Upstream release notes for results", [SRVKP-14266](https://redhat.atlassian.net/browse/SRVKP-14266) |
| A pull request in the range, with no ticket that links it | 1 | [pipeline#9682](https://github.com/tektoncd/pipeline/pull/9682), the only Pipelines entry, cites [SRVKP-7717](https://redhat.atlassian.net/browse/SRVKP-7717), which links a different pull request |
| A change that shipped in 1.23, with Jira fix version 1.24.0 | 5 | The fix version was set from the plan, months before the 1.23 branch cut, and the note followed Jira. 3 of the 5 were already in the 1.23 notes, so 1.24.0 repeats them. 2 were not, so their note came one release late |

So the check in this ADR would have listed 25 of the 30 at the 1.24.0 freeze, and the 1.23 check would have listed the other 5, because it reads the branch and not the fix version. Only 12 entries took the path the docs dashboard expects, one change with one ticket.

Meanwhile, 45 Pipelines pull requests in the 1.24.0 range carry a release note, and the published page has one Pipelines entry. 3 of the 45 have a ticket that puts them in another release. The list is in [Appendix C](#appendix-c-pipelines-changes-in-1240-that-carry-a-release-note).

### Why it matters

The note exists in the pull request. It just has no path to Jira.

Two more groups read the notes now. In Support, Sudhish Nair built a skill that checks a customer's configuration against the breaking changes and removals in the notes between two versions, so a breaking change missing from the notes is a missed warning for a customer. Andrew Thorp's team, which runs Pipelines on Konflux, reviews the change log of every upgraded component for breaking changes on each deployment, and says complete notes would save much of that work.

### Constraints

- Downstream repositories carry almost no changes of their own. Since 2026-07-01, people wrote 12 of the pull requests merged into the 19 downstream repositories, and all 12 touched only `.konflux/` build files. Bots wrote the rest, syncing upstream code. So the upstream pull request is the one place a change is described.
- Jira can require a field when a ticket is created or when it moves to a given status. The release-note fields are known only once the change exists, so this ADR requires them at a status change, not at creation. Spikes and docs work have no pull request, and CVE tickets come one per image.
- Developers set target version, and the Release Captain sets fix version once the release starts. Naomi Gelman proposed this after the 1.24.0 retrospective, and it is part of the Release Captain handbook in [SRVKP-14379](https://redhat.atlassian.net/browse/SRVKP-14379). Jira already stops developers from setting fix version.
- Only tektoncd/pipeline marks a feature as alpha, through its feature flags. In the other components a person decides whether a change is Technology Preview. This ADR does not automate that call. The owner sets Release Note Type to Technology Preview when approving, and rejects the note of an upstream addition that is not supported downstream yet. Requiring Release Note Type at Release Pending, plan item 2, makes the owner decide this for every change.

## Decision

`Required`

Jira stays the one place the docs team reads. We give each upstream note a path to Jira in four steps:

1. **At the pull request.** Every pull request carries a release note.
2. **At merge.** A bot copies the note to the ticket.
3. **At the RN Dashboard.** The owner of the change approves or rejects the note on the RN Dashboard, a new component this ADR adds, before the docs team sees it.
4. **At release time.** A check lists every change that still has a gap, and the Release Captain works from that list on the RN Dashboard.

- Chosen option: the four steps below, over a model drafting notes from the code.

### Architectural Overview

`Required`

```
upstream PR  (release-note block, CI check)
│ merge
▼
jirasync  (copies the note, sets Awaiting Approval)
▼
SRVKP ticket  ◄──────────  release-time check, in one-click-release
▼
RN Dashboard  (owner approves or rejects)
▼
docs dashboard
▼
docs team
```

1. **At the pull request.** Every pull request in an upstream repository that ships in the product carries a release-note block. A change that users will not notice says `NONE`. A CI check warns when the block is missing. One release later, the check becomes required, and a pull request without the block cannot merge. This is the only rule upstream. Tekton repositories already ask for the block in their pull request template. Nothing upstream mentions Jira, and contributors outside Red Hat never see SRVKP.

   The SRVKP ticket is Red Hat's own work. The owner of the change creates or links it and puts the pull request in its Git Pull Request field, so the link goes from the ticket to the pull request. The owner is the Red Hat engineer who wrote the change. For a change from outside Red Hat, it is the Red Hat owner of that component. They act when the release-time check lists the change as "no ticket", see step 4.
2. **At merge.** [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync), in [openshift-pipelines/skipjira](https://github.com/openshift-pipelines/skipjira), looks for the ticket whose Git Pull Request field names each merged pull request. It never creates a ticket. When no ticket names the pull request, [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) does nothing at merge, and the release-time check lists the change as "no ticket", see step 4. [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) reads only pull requests by the people on its list of team members, so a change from outside Red Hat also reaches Jira through the check. [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) cannot read pull requests in private forks, which hold embargoed security fixes. The docs team confirmed these need no release note. They go in the advisory only.

   Today [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) writes a note it finds in the pull request straight into the fields, with status Proposed. When the pull request has no block, it posts a Gemini draft as a Jira comment, [as on SRVKP-15008](https://redhat.atlassian.net/browse/SRVKP-15008?focusedCommentId=19128774), and a person copies it into the fields by hand. Under this ADR every note waits for the owner. When a ticket names the pull request, [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) copies the release-note block into Release Note Text, sets Release Note Type from the pull request's kind, and sets Release Note Status to Awaiting Approval. When a linked pull request has no block, [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) drafts the text with Gemini from the diff and writes the draft into Release Note Text the same way. Nothing reaches the docs dashboard until the owner approves it in the next step, so no Gemini draft is published unread. [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) already moves the ticket to Code Review when the pull request is ready for review, and to Dev Complete at merge. This ADR adds the field writes and no new transition.
3. **At the RN Dashboard.** Every note waits for a person who knows the change. The owner opens the RN Dashboard, reads the note, edits it if needed, and approves or rejects it. Approval sets Release Note Status to Proposed, and the ticket reaches the docs dashboard. Rejection sets Release Note Type to Not Required and the status to Rejected. A bot account makes the change and adds a Jira comment such as "Approved by Jane Doe (jdoe@redhat.com) on the RN Dashboard, 2026-10-18 14:02 UTC".

   One change can span several pull requests or several stories under an epic, such as a feature that touches pipeline, operator and cli. When they share one ticket, the owner edits their notes into one before approving. When each has its own story, the owner approves one combined note on one story and rejects the note on the others, which sets their Release Note Type to Not Required. One change gets one note, so not every story needs its own. The check counts a change as ready when any ticket that names its pull request is ready. The RN Dashboard does not yet group related "no ticket" rows, so the owner has to notice that they belong together. Grouping them is left until the first release on this process shows how often it happens.

   The owner or another engineer on the same component decides, from the owner view. The Release Captain's view has no approve or reject button, so the Release Captain chases owners and does not decide for them. A Slack app posts a daily summary of the waiting tickets to each team's channel, and sends the owner a direct message when a ticket has waited 3 days.

   Once the ticket reaches the docs dashboard, the docs team gives a second approval. A writer accepts the note, rewrites it, or sends it back to the owner with a Jira comment. Done stays the docs team's own status, set after QE review.
4. **At release time.** The release-time check extends Akshay Pant's audit tool, [`cmd/audit`](https://github.com/theakshaypant/osp-rc-tools/blob/main/cmd/audit/main.go) in [theakshaypant/osp-rc-tools](https://github.com/theakshaypant/osp-rc-tools). It moves into [one-click-release](https://github.com/openshift-pipelines/one-click-release) so the team owns it. The tool already:

   - finds each component's commit range from the downstream tags and hack's release files
   - resolves the pull requests, including the original of each cherry-pick
   - finds each pull request's ticket through the Git Pull Request field and reads its release-note fields

   The check adds:

   - an end point taken from the new release's core snapshot in Konflux, which records the commit each image was built from, because the downstream tag the tool reads today exists only after the release ships
   - a filter that leaves out pull requests opened by bots, such as dependency updates and automated backports, so they never reach the list or get a ticket. A backport counts once, through the original pull request the tool already resolves
   - one verdict per change, from the table below, with the changes that need action first
   - the list in the Release Captain's view of the RN Dashboard

   By the docs deadline, three weeks before release, each remaining change gets one of two outcomes. Either the owner marks it not required, or the docs team writes it from the pull request and the owner reviews it. The Release Captain chases owners but does not decide for them. The number still empty at the deadline goes to the release retrospective.

| Verdict | Meaning | Who acts |
|---|---|---|
| no ticket | The change needs a note and has no SRVKP ticket | The owner creates one, links an existing one, or says no note is needed, from the RN Dashboard |
| fields missing | The ticket exists, but a release-note field or target version is empty | The owner |
| awaiting approval | The note is on the ticket, but nobody has approved or rejected it | The owner, or another engineer on the component |
| ready | Every field is set and the ticket is on the docs dashboard | Nobody |
| not required | The ticket's Release Note Type is Not Required, set by the owner at any time or when they rejected the note. Or the pull request has no ticket and is on the release's no-note list | Nobody |
| CVE | A security fix. It goes in the advisory, never in the notes | Nobody |
| other release | It shipped in an earlier release | Nobody |

Fix version comes from the Release Captain with the rest of the release. A ticket approved before that becomes ready once fix version is set.

The RN Dashboard offers three actions, one click each. Create opens a new SRVKP ticket with Git Pull Request, Release Note Text and Release Note Type filled from the pull request. Link takes an existing ticket key and fills its Git Pull Request field. No note needed creates no ticket. A bot adds the pull request's URL, who decided and when to the release's no-note list, a file in one-click-release beside the check, and the check lists the change as "not required" from then on. Jira holds the notes, not the changes that need none. The bot account makes the Jira changes, as it does for approvals.

A ticket made by Create records work that has already merged, so it skips the backlog. It is a Story with the pull request's title as summary, the component mapped from the repository, the owner as assignee and the release as target version. Its parent is the release's "Upstream release notes" epic, where the team lists upstream work today. The bot moves it straight to Release Pending, filling what each transition asks for, so it never waits in New or in a sprint. The Release Captain sets fix version on it with the rest of the release, and the release closes it like any other ticket, so it reaches the advisory like the stories already under that epic.

### Implementation Details

`Required`

- **Upstream repositories.** Add a release-note check to each repository that lacks one. tektoncd/pipeline, cli, operator, triggers, results and tekton-kueue already run Prow's `release-note` plugin. Pipelines as Code, console-plugin, opc and the smaller openshift-pipelines repositories have no release-note block today. Pipelines as Code comes first: in 1.24.0 it had 46 of the 59 changes with no block.
- **Downstream repositories.** Optional. Run the same check only on pull requests that people open, so bot pull requests keep flowing.
- **[jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync).** [skipjira#30](https://github.com/openshift-pipelines/skipjira/pull/30) is merged. It stops [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) moving tickets back out of Release Pending, the bug that got it switched off. Next, [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) reads the Tekton release-note block, writes Gemini drafts into the fields instead of a comment, and sets Awaiting Approval instead of Proposed. Until that is built, it writes no release-note fields, so 1.25 runs it as a dry run. [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) never compares the text on a ticket with the text in a pull request. It keeps its own record instead: each time it copies a note, it adds a Jira comment that names the pull request. When a ticket names a pull request that has no such comment yet, [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) adds that note as a new paragraph and leaves the existing text as it is. It then sets Awaiting Approval again, so the owner sees the addition. If the docs team has set Done, [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) only adds a comment and changes nothing else. It skips backport pull requests that bots open, because they carry the same note as the original. It never changes text a person has written or approved. A ticket whose Release Note Type is Not Required counts as decided. [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) changes none of its fields, and the ticket never enters the owner's queue. The Release Captain still sees it in the "not required" group of the check's list, so a wrong call can be caught.
- **Jira.** Expect the Git Pull Request field on every ticket except CVE, spike and docs tickets. Add Awaiting Approval to the Release Note Status field, and add it to the docs dashboard's "needs attention" gadget so writers still see these tickets. Change the automation that copies fix version into target version, because it runs the opposite way to the rule above.
- **Release-time check.** Move the audit tool into openshift-pipelines. Add the range from the downstream tag to the snapshot, the verdicts and the list.
- **RN Dashboard.** One internal page with two views, one for owners to approve or reject, one for the Release Captain to see the check's list and nudge owners. People sign in with Red Hat SSO, and a Jira service account writes.

Affected components are every upstream repository in the product, [openshift-pipelines/skipjira](https://github.com/openshift-pipelines/skipjira), [openshift-pipelines/one-click-release](https://github.com/openshift-pipelines/one-click-release), the SRVKP Jira project and its automation, the [docs dashboard](https://redhat.atlassian.net/jira/dashboards/22264), the new RN Dashboard and its Slack app.

## Implementation Plan

`Optional`

The work is 20 items, 19 of them filed as issues under the epic [SRVKP-14410](https://redhat.atlassian.net/browse/SRVKP-14410). The 1.25 code freeze is at the end of November 2026. 1.25 gets the release-time check, and [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) reading the release-note block with its field writes in a dry run. 1.26 gets [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) writing to Jira, the CI checks, the RN Dashboard and the Slack app. Each item's Jira issue names its owner. Items 13 to 18 have no owner yet.

| # | Jira | Type | Work | Points | Needs | Release |
|---|---|---|---|---|---|---|
| 1 | [SRVKP-15341](https://redhat.atlassian.net/browse/SRVKP-15341) | Task | Release Note Status value and the fix version automation | 2 | - | 1.25 |
| 2 | [SRVKP-15343](https://redhat.atlassian.net/browse/SRVKP-15343) | Task | Required fields in the SRVKP workflow | 5 | - | 1.26 |
| 3 | [SRVKP-15344](https://redhat.atlassian.net/browse/SRVKP-15344) | Task | Awaiting Approval on the docs dashboard | 1 | 1 | 1.26 |
| 4 | [SRVKP-15345](https://redhat.atlassian.net/browse/SRVKP-15345) | Story | Move the audit tool into [one-click-release](https://github.com/openshift-pipelines/one-click-release) | 3 | - | 1.25 |
| 5 | [SRVKP-15346](https://redhat.atlassian.net/browse/SRVKP-15346) | Story | Build the release-time check | 5 | 4 | 1.25 |
| 6 | [SRVKP-15358](https://redhat.atlassian.net/browse/SRVKP-15358) | Task | Run the check at the 1.25 freeze | 3 | 5 | 1.25 |
| 7 | [SRVKP-15359](https://redhat.atlassian.net/browse/SRVKP-15359) | Story | [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) reads the release-note block | 3 | - | 1.25 |
| 8 | [SRVKP-15364](https://redhat.atlassian.net/browse/SRVKP-15364) | Story | [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) field writes, dry run over 1.25 | 5 | 7 | 1.25 |
| 9 | [SRVKP-15365](https://redhat.atlassian.net/browse/SRVKP-15365) | Story | [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) writes the release-note fields | 3 | 1, 8 | 1.26 |
| 10 | [SRVKP-15366](https://redhat.atlassian.net/browse/SRVKP-15366) | Story | CI check in warn mode for Pipelines as Code | 3 | - | 1.26 |
| 11 | [SRVKP-15367](https://redhat.atlassian.net/browse/SRVKP-15367) | Story | CI check in warn mode for the other upstream repositories | 5 | 10 | 1.26 |
| 12 | not filed | Story | CI check on downstream repositories | 2 | 10 | Optional |
| 13 | [SRVKP-15368](https://redhat.atlassian.net/browse/SRVKP-15368) | Spike | RN Dashboard proof of concept | 5 | - | Start in 1.25 |
| 14 | [SRVKP-15369](https://redhat.atlassian.net/browse/SRVKP-15369) | Story | RN Dashboard owner view | 5 | 9, 13 | 1.26 |
| 15 | [SRVKP-15370](https://redhat.atlassian.net/browse/SRVKP-15370) | Story | RN Dashboard ticket create and link | 3 | 2, 5, 14 | 1.26 |
| 16 | [SRVKP-15371](https://redhat.atlassian.net/browse/SRVKP-15371) | Story | RN Dashboard Release Captain view | 3 | 6, 14 | 1.26 |
| 17 | [SRVKP-15372](https://redhat.atlassian.net/browse/SRVKP-15372) | Story | Slack app | 3 | 9, 13 | 1.26 |
| 18 | [SRVKP-15373](https://redhat.atlassian.net/browse/SRVKP-15373) | Task | Deploy the RN Dashboard and the Slack app | 3 | 3, 15, 16, 17 | 1.26 |
| 19 | [SRVKP-15374](https://redhat.atlassian.net/browse/SRVKP-15374) | Task | Switch the Pipelines as Code check from warn to fail | 2 | 10 | 1.27 |
| 20 | [SRVKP-15375](https://redhat.atlassian.net/browse/SRVKP-15375) | Task | Update the Release Captain handbook and tell the docs team | 1 | 6 | 1.26 |

Total: 63 points over the 19 filed issues. The optional item 12 is not filed. Items 4 to 8 are 19 points before the 1.25 freeze, with 4, 5 and 6 one after another. Item 6 has a fallback in case 4 or 5 slips.

### 1. Release Note Status value and the fix version automation

- An SRVKP admin adds Awaiting Approval to the Release Note Status field. The field has Proposed, In Progress, Rejected, Upstream Only, Text Approved and Done today.
- The docs team confirms the new value. Kavya Verma confirmed on 2026-10-26 that Rejected means no release note is needed, as this ADR uses it, and that approval needs engineering and docs both.
- An SRVKP admin changes the automation that copies fix version into target version. Developers set target version and the Release Captain sets fix version, so the copy runs the wrong way.
- Done when both changes are live.

### 2. Required fields in the SRVKP workflow

- Jan Richtar and an SRVKP admin make Git Pull Request required when a ticket moves to Code Review, and Release Note Type required when it moves to Release Pending. Not Required is a valid answer, so a change that needs no note costs one click.
- Type cannot be asked at Code Review. [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) moves a ticket there when its pull request is ready for review, before any note exists.
- This goes into the workflow Jan is testing on stage Jira, which already requires Git Pull Request at Code Review. Open with Jan: which issue types the rule covers.
- Before the rule goes live:
  - Automation sets Release Note Type Not Required on Vulnerability tickets. They carry a dependency pull request, about 300 a release.
  - The Release Captain's bulk move to Release Pending is tested on stage Jira. In 1.24.0, 66 tickets reached Release Pending or Closed with Type empty, and the rule would stop every one of them.
- Done when the rule is live on SRVKP and a release's bulk move passes.

### 3. Awaiting Approval on the docs dashboard

- The docs team adds Awaiting Approval to the "needs attention" gadget on the docs dashboard, so writers still see these tickets. Today the gadget selects an empty Release Note Status, and the "ready" gadget selects Proposed or Done, so a ticket at Awaiting Approval matches neither.
- Done before item 18 turns on Awaiting Approval.

### 4. Move the audit tool into one-click-release

- The audit tool, [`cmd/audit`](https://github.com/theakshaypant/osp-rc-tools/blob/main/cmd/audit/main.go) in [theakshaypant/osp-rc-tools](https://github.com/theakshaypant/osp-rc-tools), lists the upstream changes in a release.
- Move it into [openshift-pipelines/one-click-release](https://github.com/openshift-pipelines/one-click-release) so the team owns it and the check builds on it. one-click-release has no Go code today, so the move brings the Go module, its GitHub and Jira packages, and a CI job.
- Move the saved GitHub and Jira data of the 1.24.0 freeze replay with it, so the check's regression test runs in the team's repository.

### 5. Build the release-time check

- The range runs from the previous release's downstream tag to the core snapshot of the new one. Reading the snapshot needs a Konflux token for kflux-prd-rh02.
- Leave out pull requests opened by bots, from a list of bot accounts kept beside the check. Today the tool skips only `dependabot[bot]`. Count a backport once, through the original pull request.
- Find each pull request's ticket through the Git Pull Request field, and through an SRVKP key in the title, as console-plugin pull requests carry one.
- Each change gets one verdict from this table. The changes that need action come first.

  | Verdict | Meaning |
  |---|---|
  | no ticket | The change needs a note and has no SRVKP ticket |
  | fields missing | The ticket exists, but a release-note field or target version is empty |
  | awaiting approval | The note is on the ticket, but nobody has approved or rejected it |
  | ready | Every field is set and the ticket is on the docs dashboard |
  | not required | Release Note Type is Not Required, or the pull request is on the release's no-note list |
  | CVE | A security fix, for the advisory only |
  | other release | It shipped in an earlier release |

- A change is ready when any ticket that names its pull request is ready.
- Define the format of the release's no-note list, a file in one-click-release, and read it. Item 15 writes it.
- Write the list as JSON for the Release Captain's view, and as a Markdown report the Release Captain can read without the RN Dashboard.
- Unit tests run on saved GitHub and Jira responses. The 1.24.0 freeze replay is a regression test.

### 6. Run the check at the 1.25 freeze

- Run the check as a GitHub Actions workflow in openshift-pipelines. The Jira, GitHub and Konflux tokens are repository secrets.
- Give the report to the 1.25 Release Captain at the freeze.
- If item 4 or 5 is not ready by the freeze, run `cmd/audit` from osp-rc-tools as it is, which already lists a minor release's tickets with no note, and give that list instead.
- Take the counts per verdict to the 1.25 retrospective.
- Decide whether the check runs once at the freeze or every week through the cycle.

### 7. jirasync reads the release-note block

- Today [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) looks for a second-level heading, while the Tekton pull request template uses a first-level one. So it misses the note the author wrote and drafts one with Gemini.
- Read the Tekton release-note block. When the block says `NONE`, write nothing.
- Add tektoncd/pipelines-as-code to [`jirasync-repos.yaml`](https://github.com/openshift-pipelines/skipjira/blob/main/jirasync-repos.yaml). It is not on the list today, though in 1.24.0 it had 46 of the 59 changes with no block.
- The list names openshift-pipelines/tekton-kueue, while hack's release files use tektoncd/tekton-kueue. Point [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) at the repository hack uses.
- [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) already writes a note it finds into the fields, with status Proposed. Once it reads the Tekton block, it would do that on every Tekton ticket. Remove the release-note field secrets from its nightly workflow before this merges, so 1.25 writes no fields. Item 9 adds them back.

### 8. jirasync field writes, dry run over 1.25

- Build the field writes this ADR describes: Release Note Text from the author's block or a Gemini draft, Release Note Type from the pull request's kind, and Release Note Status Awaiting Approval. Add the Jira comment per copied pull request, the new-paragraph rule, the guard for Done, Not Required counted as decided, and skipping backports that bots open. See Implementation Details.
- Add a dry-run mode that records what it would write on each ticket and writes nothing. The workflow's `dry_run` input today runs a test command that does not cover release notes.
- Run it over every pull request merged in the 1.25 range.
- At the freeze, compare that with what people wrote, and fix what it gets wrong.

### 9. jirasync writes the release-note fields

- Turn the writes on, with the field secrets back in the workflow.
- When the RN Dashboard is switched off, it sets Proposed only for a note the pull request's author wrote. A Gemini draft stays a comment, as today, so no draft reaches the docs team unread. Item 18 switches to Awaiting Approval.
- Each night it reports how many pull requests it linked and how many it skipped.

### 10. CI check in warn mode for Pipelines as Code

- Add a check that warns when a pull request has no release-note block.
- Pipelines as Code comes first. In 1.24.0 it had 46 of the 59 changes with no block.
- The Pipelines as Code maintainers have to agree to the check.

### 11. CI check in warn mode for the other upstream repositories

- tektoncd/pipeline, cli, operator, triggers, results and tekton-kueue already run Prow's `release-note` plugin, so they need nothing.
- Add the check to chains, console-plugin, opc, git-init, manual-approval-gate, hub, pruner, multicluster-proxy-aae, syncer-service, tekton-assist and tekton-caches, where it is missing.
- Each repository's maintainers have to agree to the check.

### 12. CI check on downstream repositories

- Optional. Run the same check on downstream repositories, only on pull requests that people open.
- Since 2026-07-01, the 12 pull requests people opened downstream touched only `.konflux/` build files, so nearly every block would say `NONE`. Do it only if that changes.

### 13. RN Dashboard proof of concept

- Timeboxed to three weeks from the day it starts.
- Answer the four questions against a sandbox Jira project and a test Slack app:
  - A Red Hat SSO login maps to the right Jira account, so the comment names the right person.
  - A Jira service account can set Release Note Status and Type and add a comment on any SRVKP ticket.
  - Red Hat's Slack admins approve an app with `chat:write`, and the lookup from a Jira email to a Slack user works for direct messages.
  - How many approvals one release asks of each team. The 1.24.0 numbers suggest about 15.
- Decide where the RN Dashboard and the Slack app run, and who runs them.
- The requests for the SSO client, the Jira service account and the Slack app go out in the first week, since they take the longest.

### 14. RN Dashboard owner view

- People sign in with Red Hat SSO.
- An owner sees the tickets waiting for them, reads each note, edits it if needed, sets Release Note Type, including Technology Preview, and approves or rejects it. When a ticket holds notes from several pull requests, the owner edits them into one before approving.
- Another engineer on the same component can decide too, for example when the owner is away. The component's engineers come from the team's list of GitHub ids, with a component column.
- Filters for my tickets and my component.
- Before it writes, it checks that the status and the text have not changed since the page loaded. If they have, it reloads instead of writing.
- A Jira service account makes the change and adds a comment that names who decided.

### 15. RN Dashboard ticket create and link

- For a change with no ticket, create opens a new SRVKP ticket with Git Pull Request, Release Note Text and Release Note Type filled from the pull request. The ticket is a Story under the release's "Upstream release notes" epic, with the pull request's title as summary, the component mapped from the repository, the owner as assignee and the release as target version. The bot moves it straight to Release Pending, filling each field the workflow asks for, so it never enters the backlog.
- Link takes an existing ticket key and fills its Git Pull Request field.
- No note needed creates no ticket. The bot commits the pull request's URL, who decided and when to the release's no-note list in one-click-release, in the format item 5 defines.

### 16. RN Dashboard Release Captain view

- The check's list for the release, grouped by verdict, read from the JSON item 5 writes.
- Who has to act on each change, and how long each ticket has waited.
- It has no approve or reject button. The Release Captain chases owners and does not decide for them.

### 17. Slack app

- A daily summary of the waiting tickets in each team's channel.
- A direct message to the owner when a ticket has waited 3 days.
- It looks up Slack users by email. If that is blocked, it uses a Slack column in the team's list of GitHub ids. If neither works, it posts to team channels only.

### 18. Deploy the RN Dashboard and the Slack app

- Deploy both where item 13 decided.
- Turn on [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync)'s Awaiting Approval mode at the same time, so owners have somewhere to approve. From then on a Gemini draft goes into Release Note Text, not a comment.

### 19. Switch the Pipelines as Code check from warn to fail

- The check warns for one release, then fails pull requests from the next.
- The epic closes when Pipelines as Code fails pull requests. Switches in other repositories are filed outside the epic.

### 20. Update the Release Captain handbook and tell the docs team

- Add to the Release Captain handbook, [SRVKP-14379](https://redhat.atlassian.net/browse/SRVKP-14379): create the release's "Upstream release notes" epic when the release starts, run the check, and work from its list.
- Tell the docs team that every note now reaches them through a ticket, and that the per-component collection of upstream notes near the end of a release stops.

## Security & Compliance

`Optional`

- [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) and the release-time check need a Jira token that can write to SRVKP and a GitHub token that can read. The check also needs a Konflux token that can read Snapshots on kflux-prd-rh02. Both run in GitHub Actions under openshift-pipelines, with the tokens stored as repository secrets.
- The RN Dashboard's bot needs a GitHub token that can write to one-click-release, to commit the release's no-note list.
- The RN Dashboard's Jira service account can write the release-note fields on any SRVKP ticket. It only acts for a person signed in with Red Hat SSO, and every write carries a comment that names that person.
- The Slack app's token can post to channels and send direct messages. It reads no messages.
- CVE numbers never go in the release notes. CVE tickets stay on the advisory path.

## Compatibility & Migration

`Optional`

- A repository's CI check can warn for one release and start failing pull requests from the next.
- Each part can be switched off on its own. With [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) or the check off, the process falls back to how it works today. With the RN Dashboard off, [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) sets Proposed instead of Awaiting Approval for a note the pull request's author wrote, and leaves a Gemini draft as a comment, as today.
- 838 SRVKP tickets have Release Note Text and an empty status. We leave them as they are. The RN Dashboard shows only tickets in a release's range, and the check lists those anyway.

## Testability

`Required`

- Claude ran the release-time check against the minor and patch releases from 1.20 to 1.24. Those releases published 360 entries. The check covers 273 of them and finds the ticket for 186. It misses 87. 26 were notes added late, 24 were changes with nothing in the pull request to go on, 20 were entries with no code change behind them, and 17 were other cases.
- The check gets unit tests that run on saved GitHub and Jira responses. The 1.24.0 freeze replay stays as a regression test.
- [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync)'s field writing runs as a dry run for one release first. We compare what it would write against what people wrote.

### Proof of concept, to be done

Before the RN Dashboard is built, a proof of concept answers four questions against a sandbox Jira project and a test Slack app.

- [ ] A Red Hat SSO login on the RN Dashboard maps to the right Jira account, so the comment names the right person.
- [ ] A Jira service account can set Release Note Status and Release Note Type and add a comment on any SRVKP ticket.
- [ ] Red Hat's Slack admins approve an app with `chat:write`, and the lookup from a Jira email to a Slack user works for direct messages. If the lookup is blocked, the app reads Slack ids from the list the team already keeps for GitHub ids, with a Slack column added. If neither works, it posts to team channels only.
- [ ] How many approvals one release asks of each team. The 1.24.0 numbers suggest about 15.

## Operational Readiness

`Optional`

- [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) reports each night how many pull requests it linked and how many it skipped.
- The release-time list is the measure. It works if the "no ticket", "fields missing" and "awaiting approval" rows at code freeze go down from one release to the next.
- The RN Dashboard shows how many tickets are waiting for approval, and how many have waited more than 3 days.

## Scope

`Required`

- **In-Scope:**
  - Getting every user-facing upstream change onto the docs dashboard with text a writer can use
  - The release-note rule and its CI check in each repository
  - [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) writing the release-note fields
  - The owner's approval on the RN Dashboard, and the Slack reminders
  - The release-time check and the Release Captain's view
- **Out-of-Scope:**
  - The final wording of the notes, which the docs team owns
  - Moving Release Note Status to Done, which the docs team owns
  - The security advisory and CVE text
  - The must-have and nice-to-have labels that decide what the writers highlight
  - The rest of release automation, which the centralised release automation ADR covers

## Acceptance Criteria

`Required`

The decision is considered accepted when:

- [ ] Problem, scope, and non-goals are clear
- [ ] Alternatives were considered and compared
- [ ] Chosen option's trade-offs (not just benefits) are documented
- [ ] The docs team agrees the docs dashboard stays the hand-off point
- [ ] Component owners agree to the release-note rule and the CI check
- [ ] An owner is named for [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync) and the release-time check
- [ ] Required reviewers approved
- [ ] Open concerns resolved or tracked

## Considered Options

`Required`

| Option | Pros | Cons |
| --- | --- | --- |
| A note on every pull request, a sync at merge, the owner's approval and a check at release time. Chosen | The person who made the change writes the note, once. Someone who knows the change checks every note before the docs team sees it, including the ones Gemini drafted. Every gap has an owner at code freeze | Each repository needs a CI check. Developers write one more block. Owners approve each note, one more click per note. The RN Dashboard and the Slack app are new and someone has to run them |
| A sync at merge straight to Proposed, with no approval | No RN Dashboard to build. No extra click | Gemini drafts reach the docs team unchecked. Nobody gets a per-team list of what is waiting |
| Owners approve in Jira itself | No RN Dashboard. Jira's history records who changed the status | Nobody gets a per-team queue or reminders. The Release Captain still has no list in one place |
| Do nothing | No work | Notes keep depending on memory, with the gaps shown in Context |
| A model drafts notes from the code at release time | Nothing changes for developers | Models miss breaking changes, the entries that matter most. A draft still needs a ticket and a person to check it |
| One release-note pull request per component repository | The notes sit next to the code | The docs team reads only Jira, so this adds a second place to keep in step |

## Decision Drivers

`Required`

- The note reaches Jira, the one place the docs team reads.
- The person who made the change writes the note, once.
- A person who knows the change approves the note before it counts as "include this entry", which is what Proposed means to the docs team.
- It builds on tools the team already runs, such as [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync), the audit tool and the Jira automation.

## Consequences

`Required`

- Positive outcomes:
  - At code freeze every user-facing change is either on the docs dashboard or on an owner's list.
  - Changes labelled action required upstream reach the notes.
  - The Release Captain works from a list instead of hunting for gaps.
- Risks:
  - Approvals can pile up. The Slack reminders and the Release Captain's view are there to catch it.
  - Owners may approve their own note from the pull request without reading it again.
  - A developer may write `NONE` on a change that needs a note. The check catches this only when a ticket exists.
  - The team may not fill the Git Pull Request field reliably. Today it is filled on about half the tickets in a release. Requiring the field at Code Review, item 2, closes the gap on tickets that exist. The larger gap, a change with no ticket at all, 110 of 155 in 1.24.0, is caught only by the check's "no ticket" verdict, and only before the docs deadline if owners act on it.
  - A note can describe an upstream feature that is Technology Preview, or not supported downstream yet. Only the owner's approval and the docs team's review catch it; nothing checks it automatically.
  - Changes with nothing in the pull request to go on stay invisible to the check, see Testability.
  - If nobody owns the tools, they can break and nobody notices.
- Follow-up:
  - Decide whether the check runs once at freeze or every week through the cycle.
  - After two releases, decide from the approval numbers whether notes the author wrote in the pull request still need approval, or only the notes Gemini drafted.

## Dependencies

`Required`

- [openshift-pipelines/skipjira](https://github.com/openshift-pipelines/skipjira), which holds [jirasync](https://github.com/openshift-pipelines/skipjira/tree/main/cmd/jirasync)
- [openshift-pipelines/one-click-release](https://github.com/openshift-pipelines/one-click-release) and the centralised release automation ADR
- SRVKP Jira fields and automation, which an SRVKP admin has to change
- The docs dashboard's "needs attention" gadget, which the docs team changes
- A Red Hat SSO client for the RN Dashboard, and Slack admin approval for its app
- hack's release files and the Konflux core snapshot, which give the range for each release

## References

`Required`

- [Centralised release automation ADR, pull request 7](https://github.com/openshift-pipelines/architecture/pull/7)
- [skipjira#30, skip transition for tickets in Release Pending](https://github.com/openshift-pipelines/skipjira/pull/30)
- [openshift-pipelines/skipjira](https://github.com/openshift-pipelines/skipjira)
- [OpenShift Pipelines release notes](https://docs.redhat.com/en/documentation/red_hat_openshift_pipelines/)
- [JIRA SRVKP-14410, Release notes from upstream pull requests to Jira](https://redhat.atlassian.net/browse/SRVKP-14410)
- [JIRA SRVKP-14379, Release Captain handbook](https://redhat.atlassian.net/browse/SRVKP-14379)

## Change Log

`Required`

| Date | Author | Change Summary |
| --- | --- | --- |
| 2026-10-19 | @psrvere | Created initial ADR |

## Appendix A. The 1.24.0 range per repository

For each component, the range runs from the upstream branch hack's release file lists for 1.23 to the branch it lists for 1.24. Every pull request merged in that range counts, except those opened by bots and those merged after 2026-09-09. That is taken as the day 1.24.0 shipped, since Jira's 1.24.0 version has no release date. A change merged after the freeze on 2026-08-28 still shipped, so it counts, while the ticket state is read at the freeze.

| Repository | Branch range | Pull requests | Author wrote a release note | No release-note block, feat, fix, perf or security title |
|---|---|---|---|---|
| tektoncd/operator | release-v0.80.x to release-v0.81.x | 114 | 29 | 0 |
| tektoncd/pipeline | release-v1.12.x to release-v1.15.x | 106 | 45 | 0 |
| tektoncd/pipelines-as-code | release-v0.48.x to release-v0.50.x | 83 | 1 | 46 |
| openshift-pipelines/console-plugin | release-v1.23.x to release-v1.24.x | 44 | 0 | 1 |
| tektoncd/tekton-kueue | release-v0.3.x to release-v0.4.x | 39 | 1 | 10 |
| tektoncd/chains | release-v0.27.x to release-v0.29.x | 36 | 12 | 0 |
| tektoncd/cli | release-v0.45.x to release-v0.46.x | 18 | 3 | 0 |
| tektoncd/triggers | release-v0.36.x to release-v0.37.x | 17 | 2 | 0 |
| tektoncd/results | release-v0.19.x to release-v0.20.x | 11 | 3 | 0 |
| openshift-pipelines/opc | release-v1.23.x to release-v1.24.x | 9 | 0 | 2 |
| Total | | 477 | 96 | 59 |

Components left out:

- git-init, manual-approval-gate, hub, pruner, the PatternFly 5 console plugin, multicluster-proxy-aae and syncer-service. Their branch is the same for 1.23 and 1.24, so nothing new shipped.
- tekton-assist and tekton-caches. The branch hack lists for 1.23 does not exist in the upstream repository.

console-plugin pull requests carry an SRVKP key in the title and rarely a release-note block. So the count finds 1 of them, while 4 console-plugin features were published. The release-time check in this ADR also reads the key and finds those.

## Appendix B. 1.24.0 entries and their pull requests

Each published entry of the 1.24.0 release notes, the upstream pull request Claude traced it to, the SRVKP ticket the check finds for that pull request, and the Jira key the published entry cites.

### A pull request in the 1.24.0 range, with its own SRVKP ticket, 12 entries

| # | Published entry | Upstream pull request | Ticket the check finds | Key the entry cites |
|---|---|---|---|---|
| 1 | Server-Side Apply for Tekton Results finalizer management | [tektoncd/results#1322](https://github.com/tektoncd/results/pull/1322) | [SRVKP-8541](https://redhat.atlassian.net/browse/SRVKP-8541) | [SRVKP-8541](https://redhat.atlassian.net/browse/SRVKP-8541) |
| 2 | Default NetworkPolicies for Manual Approval Gate | [tektoncd/operator#3801](https://github.com/tektoncd/operator/pull/3801) | [SRVKP-12052](https://redhat.atlassian.net/browse/SRVKP-12052) | none |
| 3 | MultiCluster: CEL-based routing policies in Tekton Kueue (Technology Preview) | [tektoncd/tekton-kueue#513](https://github.com/tektoncd/tekton-kueue/pull/513) | [SRVKP-12952](https://redhat.atlassian.net/browse/SRVKP-12952) | [SRVKP-12732](https://redhat.atlassian.net/browse/SRVKP-12732) |
| 4 | NetworkPolicy support for OpenShift Pipelines Operator | [tektoncd/operator#3714](https://github.com/tektoncd/operator/pull/3714) | [SRVKP-12050](https://redhat.atlassian.net/browse/SRVKP-12050) | [SRVKP-12050](https://redhat.atlassian.net/browse/SRVKP-12050) |
| 5 | Persistent date-range filter on PipelineRuns and TaskRuns list pages | [openshift-pipelines/console-plugin#1200](https://github.com/openshift-pipelines/console-plugin/pull/1200), [openshift-pipelines/console-plugin#1180](https://github.com/openshift-pipelines/console-plugin/pull/1180), [openshift-pipelines/console-plugin#1133](https://github.com/openshift-pipelines/console-plugin/pull/1133) | [SRVKP-12328](https://redhat.atlassian.net/browse/SRVKP-12328), [SRVKP-12329](https://redhat.atlassian.net/browse/SRVKP-12329), [SRVKP-13054](https://redhat.atlassian.net/browse/SRVKP-13054) | [SRVKP-12008](https://redhat.atlassian.net/browse/SRVKP-12008) |
| 6 | Persistent datasource filter on PipelineRuns and TaskRuns list pages | [openshift-pipelines/console-plugin#1215](https://github.com/openshift-pipelines/console-plugin/pull/1215), [openshift-pipelines/console-plugin#1152](https://github.com/openshift-pipelines/console-plugin/pull/1152) | [SRVKP-12524](https://redhat.atlassian.net/browse/SRVKP-12524), [SRVKP-12525](https://redhat.atlassian.net/browse/SRVKP-12525), [SRVKP-12917](https://redhat.atlassian.net/browse/SRVKP-12917) | [SRVKP-12009](https://redhat.atlassian.net/browse/SRVKP-12009) |
| 7 | Default PVC configuration for Pipeline workspaces in UI | [openshift-pipelines/console-plugin#1150](https://github.com/openshift-pipelines/console-plugin/pull/1150) | [SRVKP-12527](https://redhat.atlassian.net/browse/SRVKP-12527) | [SRVKP-12527](https://redhat.atlassian.net/browse/SRVKP-12527) |
| 8 | Option to toggle optional task parameters in Pipeline Builder | [openshift-pipelines/console-plugin#1153](https://github.com/openshift-pipelines/console-plugin/pull/1153), [openshift-pipelines/console-plugin#1154](https://github.com/openshift-pipelines/console-plugin/pull/1154) | [SRVKP-12530](https://redhat.atlassian.net/browse/SRVKP-12530), [SRVKP-12531](https://redhat.atlassian.net/browse/SRVKP-12531) | [SRVKP-12530](https://redhat.atlassian.net/browse/SRVKP-12530), [SRVKP-12531](https://redhat.atlassian.net/browse/SRVKP-12531) |
| 9 | Forgejo and Gitea ListOrgTeams API caching | [tektoncd/pipelines-as-code#2775](https://github.com/tektoncd/pipelines-as-code/pull/2775) | [SRVKP-11532](https://redhat.atlassian.net/browse/SRVKP-11532) | [SRVKP-11532](https://redhat.atlassian.net/browse/SRVKP-11532) |
| 10 | Kubernetes event timestamp population | [tektoncd/pipelines-as-code#2896](https://github.com/tektoncd/pipelines-as-code/pull/2896) | [SRVKP-12646](https://redhat.atlassian.net/browse/SRVKP-12646) | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 11 | Tekton Hub commands in tkn CLI | [tektoncd/cli#2876](https://github.com/tektoncd/cli/pull/2876) | [SRVKP-10462](https://redhat.atlassian.net/browse/SRVKP-10462) | [SRVKP-10462](https://redhat.atlassian.net/browse/SRVKP-10462), [SRVKP-11950](https://redhat.atlassian.net/browse/SRVKP-11950) |
| 12 | Tekton Hub integration in Pipelines as Code | [tektoncd/pipelines-as-code#2855](https://github.com/tektoncd/pipelines-as-code/pull/2855) | [SRVKP-11798](https://redhat.atlassian.net/browse/SRVKP-11798), [SRVKP-12188](https://redhat.atlassian.net/browse/SRVKP-12188) | [SRVKP-12188](https://redhat.atlassian.net/browse/SRVKP-12188) |

### A pull request in the range, listed by hand in an "Upstream release notes" story, 12 entries

| # | Published entry | Upstream pull request | Ticket the check finds | Key the entry cites |
|---|---|---|---|---|
| 13 | Support service accounts for Bitbucket Data Center scoped tokens | [tektoncd/pipelines-as-code#2726](https://github.com/tektoncd/pipelines-as-code/pull/2726) | none | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 14 | Direct OpenTelemetry SDK tracing setup | [tektoncd/pipelines-as-code#2756](https://github.com/tektoncd/pipelines-as-code/pull/2756) | none | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 15 | Reconciler shared state race condition | [tektoncd/pipelines-as-code#2825](https://github.com/tektoncd/pipelines-as-code/pull/2825) | none | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 16 | GitLab token rotation write access verification | [tektoncd/pipelines-as-code#2837](https://github.com/tektoncd/pipelines-as-code/pull/2837) | none | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 17 | GitLab branch creation push events | [tektoncd/pipelines-as-code#2875](https://github.com/tektoncd/pipelines-as-code/pull/2875) | none | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 18 | GitLab source lookups pinned to event SHA | [tektoncd/pipelines-as-code#2898](https://github.com/tektoncd/pipelines-as-code/pull/2898) | none | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 19 | Nil content handling in Gitea file retrieval | [tektoncd/pipelines-as-code#2894](https://github.com/tektoncd/pipelines-as-code/pull/2894) | none | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 20 | Bitbucket Cloud token CLI updates | [tektoncd/pipelines-as-code#2851](https://github.com/tektoncd/pipelines-as-code/pull/2851) | none | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 21 | Controller HTTP listener graceful shutdown | [tektoncd/pipelines-as-code#2877](https://github.com/tektoncd/pipelines-as-code/pull/2877) | none | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 22 | Provider logger consistency | [tektoncd/pipelines-as-code#2827](https://github.com/tektoncd/pipelines-as-code/pull/2827) | none | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 23 | Webhook deployment Prometheus metrics port | [tektoncd/pipelines-as-code#2895](https://github.com/tektoncd/pipelines-as-code/pull/2895) | none | [SRVKP-14264](https://redhat.atlassian.net/browse/SRVKP-14264) |
| 24 | Time-based query support for the Splunk logging plugin | [tektoncd/results#1317](https://github.com/tektoncd/results/pull/1317) | none | [SRVKP-14266](https://redhat.atlassian.net/browse/SRVKP-14266) |

### A pull request in the range, with no ticket that links it, 1 entry

| # | Published entry | Upstream pull request | Ticket the check finds | Key the entry cites |
|---|---|---|---|---|
| 25 | Compressed termination messages for Tekton Tasks | [tektoncd/pipeline#9682](https://github.com/tektoncd/pipeline/pull/9682) | none | [SRVKP-7717](https://redhat.atlassian.net/browse/SRVKP-7717) |

### A change that shipped in 1.23, with Jira fix version 1.24.0, 5 entries

| # | Published entry | Upstream pull request | Ticket the check finds | Key the entry cites | In the 1.23 notes? |
|---|---|---|---|---|---|
| 26 | Custom JSON field extraction and line formatting for Loki logging plugin | [tektoncd/results#1210](https://github.com/tektoncd/results/pull/1210) | none | [SRVKP-14266](https://redhat.atlassian.net/browse/SRVKP-14266) | Yes, as "Loki logging plugin configuration options" |
| 27 | Support for CustomRuns in Tekton Results | [tektoncd/results#1320](https://github.com/tektoncd/results/pull/1320) | none | [SRVKP-11436](https://redhat.atlassian.net/browse/SRVKP-11436) | No |
| 28 | OpenTelemetry observability migration and metric updates | [openshift-pipelines/manual-approval-gate#965](https://github.com/openshift-pipelines/manual-approval-gate/pull/965), [tektoncd/pipeline#9043](https://github.com/tektoncd/pipeline/pull/9043), [tektoncd/pipelines-as-code#2605](https://github.com/tektoncd/pipelines-as-code/pull/2605) | none | [SRVKP-11514](https://redhat.atlassian.net/browse/SRVKP-11514), [SRVKP-8534](https://redhat.atlassian.net/browse/SRVKP-8534), [SRVKP-8544](https://redhat.atlassian.net/browse/SRVKP-8544) | Yes, as "Metrics migrated from OpenCensus to OpenTelemetry". Manual Approval Gate is the only component the 1.24.0 entry adds |
| 29 | Centrally managed TLS profile support in Pipelines as Code controller | [tektoncd/pipelines-as-code#2738](https://github.com/tektoncd/pipelines-as-code/pull/2738) | none | [SRVKP-9681](https://redhat.atlassian.net/browse/SRVKP-9681) | Partly, as "Central TLS profile consistency", which lists the Pipelines as Code webhook |
| 30 | GitHub API rate limit event emission in Pipelines as Code | [tektoncd/pipelines-as-code#2715](https://github.com/tektoncd/pipelines-as-code/pull/2715) | none | [SRVKP-10951](https://redhat.atlassian.net/browse/SRVKP-10951) | No |

Every pull request in this group is on the upstream branch 1.23 was built from, and nothing in 1.24 was needed to make it work. The fix version on each cited ticket was set to 1.24.0 by hand between March and May 2026, before the 1.23 branches were cut.

## Appendix C. Pipelines changes in 1.24.0 that carry a release note

Every pull request in tektoncd/pipeline between release-v1.12.x and release-v1.15.x, the range of 1.24.0, whose release-note block says something other than `NONE`. Pull requests opened by bots are left out. The state of the ticket is as it stood at the freeze on 2026-08-28. "Action required" means the pull request carries the upstream `release-note-action-required` label.

### No SRVKP ticket, 38 changes

[#9316](https://github.com/tektoncd/pipeline/pull/9316), [#9432](https://github.com/tektoncd/pipeline/pull/9432), [#9588](https://github.com/tektoncd/pipeline/pull/9588), [#9682](https://github.com/tektoncd/pipeline/pull/9682), [#9709](https://github.com/tektoncd/pipeline/pull/9709), [#9752](https://github.com/tektoncd/pipeline/pull/9752), [#9776](https://github.com/tektoncd/pipeline/pull/9776), [#9826](https://github.com/tektoncd/pipeline/pull/9826), [#9912](https://github.com/tektoncd/pipeline/pull/9912), [#9965](https://github.com/tektoncd/pipeline/pull/9965), [#9999](https://github.com/tektoncd/pipeline/pull/9999), [#10003](https://github.com/tektoncd/pipeline/pull/10003), [#10077](https://github.com/tektoncd/pipeline/pull/10077), [#10097](https://github.com/tektoncd/pipeline/pull/10097), [#10114](https://github.com/tektoncd/pipeline/pull/10114), [#10117](https://github.com/tektoncd/pipeline/pull/10117), [#10140](https://github.com/tektoncd/pipeline/pull/10140), [#10160](https://github.com/tektoncd/pipeline/pull/10160), [#10169](https://github.com/tektoncd/pipeline/pull/10169), [#10172](https://github.com/tektoncd/pipeline/pull/10172), [#10242](https://github.com/tektoncd/pipeline/pull/10242), [#10269](https://github.com/tektoncd/pipeline/pull/10269), [#10272](https://github.com/tektoncd/pipeline/pull/10272), [#10273](https://github.com/tektoncd/pipeline/pull/10273), [#10315](https://github.com/tektoncd/pipeline/pull/10315), [#10324](https://github.com/tektoncd/pipeline/pull/10324), [#10338](https://github.com/tektoncd/pipeline/pull/10338), [#10362](https://github.com/tektoncd/pipeline/pull/10362), [#10392](https://github.com/tektoncd/pipeline/pull/10392), [#10403](https://github.com/tektoncd/pipeline/pull/10403), [#10418](https://github.com/tektoncd/pipeline/pull/10418), [#10421](https://github.com/tektoncd/pipeline/pull/10421), [#10422](https://github.com/tektoncd/pipeline/pull/10422), [#10431](https://github.com/tektoncd/pipeline/pull/10431), [#10443](https://github.com/tektoncd/pipeline/pull/10443), [#10480](https://github.com/tektoncd/pipeline/pull/10480), [#10487](https://github.com/tektoncd/pipeline/pull/10487), [#10691](https://github.com/tektoncd/pipeline/pull/10691).

[#9588](https://github.com/tektoncd/pipeline/pull/9588) and [#10169](https://github.com/tektoncd/pipeline/pull/10169) carry the `release-note-action-required` label. Only [#9682](https://github.com/tektoncd/pipeline/pull/9682) was published.

### Ticket exists, release-note fields or version empty, 4 changes

| # | Pull request | Title | Release note in the pull request | Action required | Ticket at the freeze | Published |
|---|---|---|---|---|---|---|
| 39 | [#10007](https://github.com/tektoncd/pipeline/pull/10007) | fix issue #8255 - seperating step level and task level validation. Va… | Fix validation error when a Task uses both spec.results and spec.steps[].results in the same step script |  | [SRVKP-6276](https://redhat.atlassian.net/browse/SRVKP-6276) (missing: 1.24.0 version, type, status) |  |
| 40 | [#10050](https://github.com/tektoncd/pipeline/pull/10050) | Fix #7756 - Validate variable references in Pipeline task params | Pipeline validation now rejects invalid variable references like $(new_image) in task parameters with a clear error message, instead of silently accepting them ... |  | [SRVKP-4265](https://redhat.atlassian.net/browse/SRVKP-4265) (missing: 1.24.0 version, type, text, status) |  |
| 41 | [#10076](https://github.com/tektoncd/pipeline/pull/10076) | fix: preserve previous condition context when TaskRun is cancelled or times out | Preserve previous TaskRun condition context (reason + message) when a TaskRun is cancelled or times out due to a PipelineRun timeout, so diagnostic information ... |  | [SRVKP-12123](https://redhat.atlassian.net/browse/SRVKP-12123) (missing: 1.24.0 version, type, text, status) |  |
| 42 | [#10326](https://github.com/tektoncd/pipeline/pull/10326) | feat: add configurable grace period for transient CreateContainerError | Add `default-create-container-error-timeout` configuration option in `config-defaults` to provide a grace period before failing TaskRuns on transient ... |  | [SRVKP-12221](https://redhat.atlassian.net/browse/SRVKP-12221) (missing: 1.24.0 version, type, text, status) |  |

### Ticket says it shipped in another release, 3 changes

| # | Pull request | Title | Release note in the pull request | Action required | Ticket at the freeze | Published |
|---|---|---|---|---|---|---|
| 43 | [#9366](https://github.com/tektoncd/pipeline/pull/9366) | fix: set default resource requirements on internal containers | Internal Tekton containers (prepare, place-scripts, working-dir-initializer, sidecar-tekton-log-results) now include minimal resource requirements (requests ... |  | [SRVKP-1801](https://redhat.atlassian.net/browse/SRVKP-1801) (Pipelines 1.23.0) |  |
| 44 | [#9625](https://github.com/tektoncd/pipeline/pull/9625) | fix: respect per-resolver TTL override in cache | fix: resolver cache now respects per-resolver TTL values set in individual resolver ConfigMaps (e.g., bundleresolver-config, git-resolver-config), instead of ... |  | [SRVKP-10234](https://redhat.atlassian.net/browse/SRVKP-10234) (Pipelines 1.21.3) |  |
| 45 | [#10429](https://github.com/tektoncd/pipeline/pull/10429) | Fix PipelineRun stuck in ResolvingTaskRef when ResolutionRequest enqueue is missed | Fix PipelineRun remaining stuck in ResolvingTaskRef when a ResolutionRequest completion event is missed by periodically requeueing while remote resolution is ... |  | [SRVKP-12133](https://redhat.atlassian.net/browse/SRVKP-12133) (Pipelines 1.21.4, Pipelines 1.22.5, Pipelines 1.23.2) |  |

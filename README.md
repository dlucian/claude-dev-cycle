# claude-dev-cycle

My daily flow for shipping code I do not read line by line. One task, from an
idea to a merged pull request, checked at every step by something that did not
produce the previous step.

This is the Claude Code version, named for the harness it runs under rather than
for the idea, because the idea outlives the harness. Nothing in the method
depends on Claude Code. Three of the scripts here do.

The repository is the honest version of a written workflow. A post is a snapshot
and says so; a repository that moves when my flow moves is a claim you can check
against its commit log. If the last commit here is old, that is the useful
signal, not a broken promise.

## The stack

| Role | What I use |
|---|---|
| Plan, implement, verify | Claude Code, headless, one worker per task |
| Plan review and code review | Codex, from the other provider |
| Final check on the merge commit | GitHub Copilot review |

The model tier matters more than the model name, and the names turn over every
few months. Plan and verify on the frontier tier, implement on the cheap one.

## What is in this repo

Three gates from the cycle below, each enforcing one rule that prose could not
enforce on its own. They are small, they only need `git` and the GitHub CLI, and
they do not know anything about my machines.

| Script | The rule it enforces | Step |
|---|---|---|
| `bin/pr-open` | Open the pull request after the review rounds, not before | 7 |
| `bin/pr-release-plan` | A merged pull request is not a deployed one | 9 |
| `bin/pr-demo-gate` | No demo, no merge | 10 |

Each carries the incident that produced it in its header comment. Read those
before you decide whether the rule is worth adopting: the rule is only as good
as the failure it came from.

## Install

```sh
git clone https://github.com/dlucian/claude-dev-cycle
export PATH="$PWD/claude-dev-cycle/bin:$PATH"
gh auth status     # the scripts use your own gh credentials, and nothing else
```

Try them read-only against a pull request of your own:

```sh
pr-release-plan owner/repo 123
pr-demo-gate    owner/repo 123
```

Both print what they checked and exit non-zero only when they block. `pr-open`
is the one that writes: it pushes the current branch and opens the pull request.

## The cycle

```
plan -> review the plan -> implement -> verify against the plan
     -> mutation sweep -> code review (capped) -> open the pull request
     -> final review -> release plan -> demo -> merge
```

The pull request opens late, at step seven. That placement is deliberate and the
reason is money. See "Open the pull request last".

## 1. Plan before you write code

Spend the frontier model here, not on the implementation. A plan a model did not
have to invent produces better code than the same model discovering the shape
halfway through and patching what it already wrote.

A plan names the approach, every file it changes, the acceptance criteria, and
the demo that proves the feature works from outside. Write the acceptance
criteria into the task itself before you dispatch the planner. A planner reads
the task once, at launch; an addendum posted afterwards is invisible to it.

## 2. Review the plan

Run Codex over the plan, as though it were code, and keep it on the other
provider: a model reviewing a plan it wrote itself mostly agrees with it. Plan
review and code review catch different defect classes, so run both:

- Plan review finds design defects. Missing guards, races between components,
  wrong abstractions, cleanup paths nobody thought about.
- Code review finds implementation defects that prose cannot express. A plan can
  say "re-check the status" and the diff can still implement it as a check
  followed by a separate, non-atomic write.

One task of mine had two design races caught at plan review and folded in. The
implementation followed the folds correctly, and code review then caught a third
defect that would have shipped a money bug. Neither pass would have found the
other's.

**Fold the findings into the plan itself.** Rewrite the plan so it reads as one
coherent document. Never leave "the plan says X, actually do Y" seams, and never
post the corrections as a second document: the implementer reads one plan, and
the one it reads must be the final one.

**Sweep every section when a fold changes a fact.** If a fold renames a
parameter, grep the whole plan for every phrasing of it. One stale copy in the
acceptance criteria is enough to make an implementer build the wrong thing while
looking like it followed the plan.

## 3. Implement from the plan

Following a plan is close to mechanical, so it does not need a frontier model. A
cheaper model is enough, and the cost difference is most of what makes running
the rest of these gates affordable.

Give the implementer one task, a disposable working tree, and no permission to
push or open a pull request. Mine is a headless `claude -p` worker in a checkout
of its own. It implements, it says whether it succeeded, it exits.

## 4. Verify the implementation against the plan

Before any expensive review, run a frontier model read-only over the diff and the
plan together, and ask two questions: does this implement the plan, and did it
add anything on the way? I use a second Claude Code pass for this, with no write
tools, so it cannot start fixing what it finds.

This is the cheap pass. It catches plan-to-diff divergence, vacuous tests,
missing boundary cases, docstrings that promise what the code does not do, and
documentation the change was supposed to update. Folding those first means your
expensive reviewer spends its rounds on correctness and security instead.

## 5. Mutation-sweep every new assertion

**A test that passes proves nothing about whether it can fail.** Before a fix
ships, break the mechanism it added, run the suite, and confirm that the specific
assertion written for it goes red. Then restore the code and confirm the suite is
green again.

Put the result in the pull request as a table: what was broken, what failed.

This is the gate that pays for itself fastest. In one day it caught four tests
that described their fix instead of catching its removal:

- A harness printed `pass=61 fail=0` while three of its cases graded nothing.
  Two reached into the code by line numbers that no longer covered the function.
  One used `cd` to fix a relative path, but `cd` does not change the script's own
  `$0`, so it printed "No such file or directory" and still reported a pass.
- A test wrote its fixture in the new format, so reinstating the old format still
  passed. Neither string is a substring of the other.
- A locale test used `e` plus a combining accent, which is two characters even
  under UTF-8, so it tripped the length cap on its own and passed with the fix
  reverted.
- A concurrency test asserted one write and at most one close, which two runs
  crashing before they write anything also satisfy.

Every one of them failed in the direction that looks like success. A red suite
gets investigated. A green one closes the question.

Three sub-rules cover most of the family:

1. **A harness that cannot reach the code under test must not print a verdict
   about it.** Where a test extracts or reconstructs part of the thing it checks,
   assert the extraction worked first, and bound it by content, never by line
   numbers.
2. **Assert positively.** `got != the_wrong_value` is satisfied by the empty
   string, so a harness that produced nothing reads as a pass. Name the value you
   expect.
3. **Assert success, not counts.** "Exactly one write happened" is also true when
   every attempt died before writing.

**A mutation that reports no coverage is a claim too.** A green suite after a
mutation means either the assertion does not cover the mechanism or your edit
never touched the mechanism. Both print the same log. Confirm the edit landed
where you aimed it before you record a gap, or you send the next round after a
defect that does not exist.

**Run the sweep before the first review round, not after the last.** A gap found
before review folds together with the review's findings, in one round. Found
after, it costs a round of its own.

## 6. Code review, capped

Run Codex over the diff, read-only. Ask for every issue with a severity, a file
and line, and a short explanation, and tell it not to propose fixes. Reviewers
that fix as they go produce longer output and worse findings.

Two mechanical notes, because both cost me an afternoon. Codex's own `review`
subcommand has a hard cap of about three findings, so inline the diff into
`codex exec` and ask for all of them. And `codex exec` reads its prompt argument
and standard input, so in any background or piped context it waits forever for an
end-of-file that never comes. Redirect from `/dev/null` and the mysterious
multi-minute hangs stop.

**Cap the rounds at two, and make the cap code rather than a paragraph.** I wrote
the cap down in three separate documents and then ran ten rounds on one task,
thirteen on another, eleven on a third. The code stopped changing around round
five; the later rounds were prose and third-priority nits. A rule you have to
remember is not a rule. A script that refuses round three is.

What the cap needs to do when it refuses:

- Print the fork: merge with the rest filed as follow-up issues, run one more
  round scoped to a named blocker, or close and re-plan.
- Refuse to re-review an unchanged commit, which can only repeat the last verdict.
- Say "no blockers left, these are follow-up issues, not another round" whenever
  a round returns only lower-severity findings, and file them as issues rather
  than leaving that to willpower.
- Not spend a round on a crashed reviewer. A run that returned no verdict is not
  a review.

**Scope a fix dispatch by finding, not by filename.** Write "do not touch files
unrelated to this finding", never "do not touch any file except this one". Many
findings name the same pattern in a sibling file, and a literal implementer will
fix the file you named and skip the sibling, buying another round.

## 7. Open the pull request last

Continuous integration and hosted review bots trigger on pull request. Copilot
re-reviews the full diff on every push, and it bills the full diff every time. If
you open the pull request before the review rounds, every fold commit spends
build minutes and review budget grading code no reviewer has read yet.

Opening late gets you three things: the pull request is not a log of ten review
commits, its first build runs on post-review code, and the final reviewer reviews
once, at the end.

This looks like a preference about pull request tidiness right up until you read
it as a cost control, which is what it is.

## 8. A final reviewer on the merge commit

Whatever runs last, make sure it ran against the commit you are merging. A review
whose commit id is not the current head is stale, and a stale approval is the
easiest thing in this whole flow to mistake for a fresh one.

Two things worth checking rather than assuming:

- That the reviewer was actually requested. Requesting a Copilot review with a
  bot account's token returns a success status and does nothing, because the
  entitlement belongs to a person's account rather than to the token. Read the
  requested reviewers back rather than trusting the status code.
- That the review finished. A review in a pending state that never finalizes is
  not a review.

## 9. A release plan, because merged is not deployed

One of my pull requests added an optional environment variable, documented it in
three places, and cleared a verify pass, four review rounds, a final review, my
own checks, the build and the merge. It shipped with half the feature silently
inert in production because nobody set the variable.

No reviewer can see that class of gap. It lives in the space between "the tests
pass" and "the operator did the thing". So every pull request that adds an
environment variable, a migration, or a manual step says so in its body, derived
from the diff rather than from memory.

Then verify the running process, not the configuration file. Environment files
are read when a container is created, so editing one and restarting can leave the
process without the variable. Ask the process what it has.

## 10. Demo, or you do not know

**Post the demo on the pull request. Always.** Not in chat, not in a terminal
you will close. The pull request is the record.

A demo is external evidence: a real HTTP call for an API change, a real command
for a tool, a screenshot for a user interface. It is not test output. Tests prove
the code does what it says. A demo asks the outside world whether the assumption
behind the code was ever true.

That distinction cost me a feature. A lookup against a government tax API cleared
three review rounds, three verify rounds, thirty-six new tests and a detailed
deployment section, merged clean, and shipped completely dead. Its parser
required a field that the API does not send, and every fixture invented that
field, so the suite was green about a response shape that has never existed. One
real call would have shown it on day one. The pull request had zero comments,
because nobody ran a demo.

Two more rules that came out of demos going wrong:

- **If the feature generates text with a model, the demo uses real output.**
  Never a hand-written sample that looks like the output. Illustrative examples
  read as real and hide exactly the prompt and format bugs the real call exposes.
- **Confirm sandbox before a write demo.** Read-only probes against live systems
  are fine. Writes are not, until you know what you are writing to. Side effects
  outlive a create-then-delete; plenty of systems keep the audit row.

## Running the agents

Some of this is specific to agents rather than to code, and it is the part that
took longest to learn.

**Nudge a silent agent with the single word "continue".** Nothing else. Asking
"are you stuck, what phase are you on?" switches it from doing the work to
reporting on the work, which pollutes its context and costs a turn.

**Check the local repository for progress, and GitHub for dependency state.**
An agent working on a branch pushes nothing for the whole implement and review
phase, so GitHub shows no activity while it is perfectly healthy. Recent local
commits mean alive. Conversely, never write "blocked on task X" off a local
branch name; ask GitHub whether X is actually still open.

**Read the transcript before you kill anything.** An agent that looks stalled has
often finished and is waiting. Killing it throws away context that costs real
time to rebuild.

**Never end a turn on a dispatch with no watcher.** Name the thing that will
re-invoke you for every job still running. A dispatch nobody is watching is a
dropped chain, not a pending one: the job finishes, nothing reads the result, and
the task sits in its last state with no error and no notification. One of mine
sat for eighty minutes and restarted only because a human asked how it was going.

**An agent finishing is not the cycle closing.** Before you end a session, sweep
every agent that reported completion and confirm each one has a pushed branch, an
open pull request, and a review started. A commit sitting local-only overnight
looks identical to a finished task. The trap is a session that ends on a high
note from a different task, which is exactly when a just-finished one gets
orphaned.

**Compact between tasks, never clear.** Clearing evaporates the repository
knowledge the agent built up, which is most of what makes the second task on a
codebase cheaper than the first.

**Match the repository's exact build gate before you push.** Not the toolchain
you assume it uses. One repository of mine uses a different linter, a different
static analyzer, and a hard line-length limit, and running the wrong formatter
there catches nothing the build enforces. Four round-trips taught me to grep the
build configuration and run that exact command once, locally.

## Write the decisions down

When you overrule a review finding, write one line into a decisions file in the
repository: the tradeoff and the reasoning. Without it the reviewer raises the
identical flag on every future pull request in that repository, and each round
costs a real budget slot.

The same file is where a design invariant goes. Fold agents optimize for a green
suite, and they will quietly undo an invariant a recent pull request established
if no assertion pins it. Pin it with a test, and record why it exists.

## What it costs

Two subscriptions, not metered billing: one plan for Claude Code, which writes,
and one for Codex, which reviews. Copilot comes with GitHub. The expensive model
appears only where it pays, which is the plan and the verify pass and nothing
else.

The measurable win is not speed. It is that the review rounds got short enough
that the output is good enough to ship without reading every line.

## Adapting this

Do not configure these, port them. They are short on purpose, every assumption
is stated in the header comment, and your agent can rewrite one faster than I
could add a flag for your case. Something like:

> Read `bin/pr-release-plan`. My repos use Alembic under `db/versions/`, my
> deploy steps live in a `## Deploy` section rather than `## Release plan`, and
> I use GitLab rather than GitHub. Rewrite it for that, keep the affirmative
> naming check exactly as it is, and tell me which of its assumptions did not
> survive the move.

That last clause is the one worth keeping. An agent will happily port a script
and quietly drop the part that made it work.

The same applies to the cycle itself. It is written as prose rather than as
configuration because the point is the ordering and the reasons, and both of
those survive a change of tooling in a way that a config file does not.

## What is not here

The glue. The dispatcher that runs a headless worker per task, the fold queue,
the plan queue and the wrappers around the two reviewers add up to about 33,000
lines across 36 files, and their assumptions about my servers, checkouts and
issue conventions run through a dozen of those modules. Extracting a generic
core from that is a project, not a cleanup, and publishing a half-ported version
that does not run would be worse than publishing nothing.

So this repository is three gates and the method. The rest is described honestly
and kept private. If that changes, it changes here first.

## A note on the guard

`.github/workflows/guard.yml` checks for classes of leak, absolute home paths,
tailnet names, private addresses, rather than for a list of host names. A
deny-list of the names you are hiding publishes those names. Keep name-specific
checks in the private repository that already has the names in it.

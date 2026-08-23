---
name: delegate-to-muse
description: Use when the user explicitly asks for repository work to be handed to Muse — "delegate this to muse", "have muse do it", or the slash command. Covers workspace trust, writing the handoff, running `muse exec`, reviewing the diff that comes back, and committing it. Never reach for it unasked: work you could do directly, do directly. Not for use from inside Muse unless the user asks for nested delegation.
---

# Delegate to Muse

Muse Code is Meta's terminal coding agent. The orchestrator owns task selection,
decomposition, final review, Git history, and user communication. Muse owns inspecting the
repository, choosing implementation details, editing its bounded scope, and running
validation.

**Only delegate when the user asked you to.** A task being large, tedious, cross-cutting, or
precisely specified is not a trigger — those you do yourself. Once the user has asked, the
delegation covers the piece of work under discussion: follow-up runs, corrections, and the
commits that finish it, until they say otherwise. You do not re-ask per run.

Verified against **Muse Code 0.2.1**. If a flag is rejected, read `muse exec --help` and adapt
to the installed version rather than guessing at a replacement safety flag.

## Decide whether to delegate

Delegate work that benefits from sustained repository context: debugging, multi-file
changes, migrations, refactors, test-backed implementation, substantial investigation.

Do it directly when it is a trivial edit, a one-command lookup, or smaller than the overhead
of composing, running, and reviewing a second agent call.

Split a large request into independently reviewable outcomes, not file-sized fragments. Keep
a vertical change together when its schema, implementation, tests, and docs must agree. Run
dependent tasks sequentially. Run independent tasks concurrently only in separate worktrees;
never let two Muse processes edit one working tree.

A single coherent run beats several context-starved calls. Do not split merely because the
user said "split the task."

Do split when a run has been **killed for running too long** — that is an operational fact
about your harness, not a preference. Split along outcomes that stand alone, tell each run
which half is not its job, and name the files the other half owns so the boundary cannot blur.

The cleanest split of all is **peeling a refactor off the feature that needs it**. When a
feature needs existing code reshaped first, send the reshaping as its own run with the
acceptance bar "every existing test passes with no edits to any test file". That is a
mechanical pass condition that needs no judgment from you at review time, and it hands the next
run a validated seam instead of asking one run to refactor and build at once. A 324-line
controller became a service this way, and checking it took one diff of the moved block against
`git show HEAD:<file>` to confirm it was byte-identical.

Splitting for **breadth** rather than for a kill is the other good reason, and it has a price:
every run cold-starts and re-reads the repository, so three runs over one feature pay for
three explorations. Buy that only when the surface is genuinely wide — a backend layer, then
the UI on top of it, then polish — and make it pay by running them **sequentially in one
worktree, committing between each**. The commit does the work: it keeps each diff reviewable
on its own, lets a bad run be discarded with `git reset` without taking the good ones with it,
and hands the next run a tree whose earlier half is already validated.

## Select the model and effort

Use **`muse-spark-1.2-contributor`**. It is the configured default and it is cheap. Do not
switch models without being asked.

`muse-spark-1.2` and `muse-spark-1.1` also resolve on this account; nothing else does. An
unavailable model id is **not** caught at argument-parse time — the run starts, then dies with
`agent loop failed: model failed: model <id> does not exist or you lack access` and exit 1.

Reasoning effort is a separate axis (`none|minimal|low|medium|high|xhigh|ultra`, default
`high`). Leave it at `high` for anything with judgment in it; drop to `medium` for mechanical
work. Pass it explicitly so nothing is inherited silently, and do not ask the user to pick
effort per call.

A detailed plan does **not** make a task mechanical. Judge by the cost of a subtle mistake,
not by how precisely you wrote the prompt.

Judge **breadth** separately from difficulty. A wide task invites any model to scope itself
down silently: you get correct work covering less ground, not wrong work. On broad surfaces,
narrow the run rather than hoping.

## Establish the workspace

Before invoking:

1. Resolve the exact repository or worktree path.
2. Read the repo's instructions and release/deployment policy.
3. Snapshot `git status --short --branch`; record pre-existing edits and untracked files.
4. Define one bounded outcome, its acceptance criteria, and who owns commit and push.
5. Identify destructive, external, privileged, or deploy-triggering actions that stay outside
   Muse's authority.
6. If the change adds to an **enumerable set** — routes, nav entries, status values, error
   codes, tool lists — find every assertion that counts or lists that set *before* writing the
   prompt, and name those files in it as authorized to update.
   `grep -rn "toHaveLength([0-9]" <test dirs>` plus a grep for the current count both as a
   numeral and spelled out in prose finds most of them. Miss one and the run ends with a
   handful of failures it correctly refuses to touch, costing a round trip.

   Registries that live in **config** are the same trap in different clothes, and they cost
   more, because they fail at the lint or architecture step rather than the test step, once the
   implementation is already written. An architecture linter with a per-class exception list, a
   DI manifest, a codegen allow-list. The tell is that every existing sibling of the file you
   are about to add already appears in some config file. Open it and read the siblings before
   you send, then name it in the prompt as authorized to update.
7. Separate external actions that must stay outside Muse's authority — deploys, pushes,
   publishing — from ones the **repo's own documented workflow requires**, such as a codegen
   or translation script that calls a paid API. Muse stops and asks before the second kind.
   That is correct of it and still a wasted round trip, so either state in the prompt that the
   step is the repo's documented workflow and is authorized, or run it yourself afterwards.

### Workspace trust is the failure mode unique to Muse

Muse loads a repository's `AGENTS.md`, project skills, and hooks **only if the workspace is
trusted**. An untrusted run still succeeds, still edits files, and still reports confidently —
it simply never saw your project rules.

Trust comes from `~/.config/muse/trust.json`, or per-run from `--trust-workspace`, or
implicitly from `--yolo`.

Muse does warn, on stderr:

```
muse: warning: rules file at <path>/AGENTS.md exists, but the workspace is untrusted,
so it is skipped for this session; restart with --trust-workspace to load project rules
```

Muse's stderr is small — a couple of hundred bytes on a routine run — so read it every time
rather than tailing it. A run whose stderr lacks a `workspace trust: trusted` line did not
have your project instructions.

### Foreign personal context

Muse also loads the **user's Claude configuration** — `~/.claude/CLAUDE.md` and the Claude
skill catalog — as "foreign personal context." This is usually what you want, and it means
personal preferences carry across without pasting them. It also means Muse may cite Claude
skills that it cannot actually run. Pass `--no-foreign-personal-context` when you need a run
isolated from that, such as reproducibility testing.

Do not paste `AGENTS.md`, user config, or skills into the prompt. Fix trust instead.

A clean worktree is worth creating before you start — it makes "every change in the diff is
Muse's" true, which is what lets you review by diff alone. If the scope overlaps dirty files,
say so in the prompt or isolate the work. Muse has worktrees built in: `-w create` with
`--worktree-base <ref>`, or `-w existing --worktree-existing <path>`.

## Write the handoff prompt

Give Muse the intent and everything that affects correctness. Then get out of the way on
implementation detail — it should inspect the repo and match local patterns.

**Separate three kinds of input explicitly**, because they carry different authority:

- **Verified external facts** — a spec revision, a changed API, a fetched doc. Muse cannot
  derive these from the repo and **its training data may contradict them**. Label them
  authoritative and say *"do not correct these from memory."* Without that line, a model will
  quietly "fix" a correct new field name back to the one it remembers.
- **Settled user decisions** — say "do not revisit these."
- **Your design sketch** — label it as *shape suggestions*, subordinate to what the repo does.

**Pass conclusions and coordinates, not transcripts.** If you explored the repository before
delegating, do not paste the survey in. Muse re-reads the repo anyway and is good at it, so a
pasted survey is paid for twice and is liable to be stale by the time it is read. Carry across
only what Muse would have to be lucky to find: the exact `path/file.php:line` where the defect
lives, the name of the helper that already solves the adjacent problem, the invariant that is
true but written down nowhere. One line of coordinates beats a page of survey.

Template — omit empty sections:

```markdown
You are the implementing agent for one bounded repository task.

Goal:
<The finished behavior or artifact, not activities. Include the acceptance bar.>

Context:
<Requirements, issue details, observed errors, evidence.>

Verified external facts — AUTHORITATIVE, do not correct from memory:
<Spec/API facts checked this session, with exact names and values.>

Decisions already made — do not revisit:
<Settled choices, so they are not relitigated.>

Design (shape suggestions — follow the repository where it differs):
<Layering, key signatures, sketches.>

Scope and constraints:
- Work only in the current repository/worktree.
- Read and follow all applicable AGENTS.md and repository instructions.
- Inspect the existing implementation and follow established local patterns.
- Preserve pre-existing user changes: <paths, or "the worktree is clean">.
- Do not modify <out-of-scope areas>; report the need instead.
- Do not commit, push, deploy, publish, or perform external writes.
- Resolve ordinary implementation details autonomously from repository evidence.
- If a material ambiguity would change public behavior, data safety, or scope, stop and
  explain rather than guessing.

Tripwires — existing invariants that must not break:
- Do NOT edit existing tests to make them pass. A failing existing assertion means the
  implementation is wrong.
- <Named fragile contracts: exact strings, counts, serialized shapes, orderings.>

Acceptance criteria:
- <Observable criterion 1>

Validation:
- Run <specific commands>. Report each command, its result, and anything not run.
- Do not leave scratch or throwaway test files in the repository.

Return a concise summary of the implementation, files changed, validation results, and
remaining risks or questions. If you run out of room, stop and say plainly what you did not
build — a truthful partial report is more useful than a tidy one.
```

That last sentence earns its place: it converts a silent shortfall into a review item.

Naming the tripwires is the highest-leverage part of the prompt. "Do not edit existing tests
to make them pass" plus a list of the exact fragile contracts prevents the failure mode where
the suite goes green because the assertions moved.

That rule needs a carve-out, or it backfires. When your change legitimately alters an existing
expectation — a new route added to an asserted list, a count going from 47 to 63 — say so
explicitly and say where: *extending an expected list, or updating a count, to describe
deliberate new behavior is correct; deleting a case or relaxing `toEqual` into `toContain` is
not.* Left implicit, you get either a red suite it refuses to touch or a quietly weakened
assertion.

**Read your own constraints for self-contradiction before sending.** The expensive failure is
not a vague prompt, it is a prompt that forbids the only way to do what it demands. Before
sending, check each "do not" against each "must".

Ask for diagnosis only when diagnosis was requested; ask for implementation and validation
when a change was requested. State each constraint once. Do not add "think step by step" or
generic coding advice.

## Invoke Muse

```bash
muse exec \
  --yolo \
  --model muse-spark-1.2-contributor \
  --reasoning-effort high \
  --session-id "$SID" \
  --prompt-file /unique/path/prompt.md \
  > /unique/path/result.md 2> /unique/path/err.log
```

Run it from the repository directory. `muse exec` has no `-C` flag: set the working directory
with your shell, or root policy-gated workspace tools elsewhere with `--workspace <path>`.

**There is no `-o` flag. Stdout is the result** — redirect it or lose it.

`--yolo` disables approval prompts and the shell sandbox and trusts the workspace for that
run. It is the right flag for unattended implementation in a repository the user already
works in. For pure inspection, drop it and pass `--trust-workspace` alone: the sandbox stays
on, project rules still load, and read-only work completes fine.

Generate `SID` yourself with `uuidgen` before the run. Muse accepts any UUID for
`--session-id` and reuses that session on a later `muse exec` with the same id, so you can
plan a correction round trip without parsing a session id out of the output. `muse resume` is
the interactive TUI path and is not for headless use.

Write the prompt to a uniquely named file and pass `--prompt-file`. Never interpolate
user-controlled text into the shell.

Other flags worth knowing:

- `-w create --worktree-base HEAD` for an isolated worktree; `-w existing --worktree-existing <path>` to reuse one.
- `--json` emits a durable JSONL event stream. Events are discriminated by `payload_type`; the
  session UUID is `stream.id` where `stream.kind` is `session`, and assistant text arrives as
  `run.output.delta`. Only worth it when a program consumes the events — it is far noisier
  than the plain-text result.
- `--max-model-steps <N>` caps runaway loops.
- `--disable-web-tools` for an offline-only run.
- `--image <path>` attaches a screenshot, repeatable.

## Run it and confirm it actually ran

Long runs exceed foreground tool timeouts. Prefer background execution with the exit status
written where nothing can overwrite it:

```bash
{ muse exec ... > result.md 2> err.log; echo $? > exit.code; }
```

**Never append `; echo "EXIT=$?"` or any trailing command to the invocation.** The trailing
command's own success becomes the command's exit status, so a failed run is reported to your
harness as exit 0.

Before reviewing anything, confirm all three:

1. `exit.code` is `0`;
2. `result.md` exists and is non-empty;
3. `git status` differs from your snapshot (or the task was genuinely read-only).

Muse fails loudly, which is a real advantage over agents that exit silently: a bad model id,
an auth problem, or a tool error produces exit 1, empty stdout, and one clear line on stderr.
Read `err.log` in full — it is small. Check it for the untrusted-workspace warning even on a
successful run.

**A killed run is a different case, and re-running it blindly wastes the work.** When your own
harness stops a long background run, no exit-code file is written at all — absent, not
nonzero. Inspect before deciding: `git status` against your snapshot, `wc -l` the new files
for truncation, and the tail of stderr. A clean worktree means it was still reading and
nothing is lost; re-run, ideally narrower. A worktree full of complete files means it was
killed while emitting its summary, and the work may be finished. Validate what is on disk
before throwing it away, and finish small gaps yourself rather than spending another round
trip.

For concurrent runs: a distinct session id, prompt file, result path, and log per task; a
separate worktree per editing process; wait on every process and check every exit status.

The trailing `Shell cwd was reset to <path>` line on stderr is cosmetic. It reports that Muse
restored a shell working directory after the run and says nothing about success. The result
file is sometimes cosmetically odd too: on longer runs it can hold the same summary twice, two
near-identical reports concatenated. Know that before you read the second copy hunting for what
changed between them.

## Review

Muse's summary is a claim, not evidence. Verify independently:

1. Diff `git status` against the pre-run snapshot; note files it touched that you did not scope.
   Muse runs code to check its own work, so expect untracked build artifacts —
   `__pycache__/`, coverage files, a stray log — beside the real changes. They are evidence it
   validated rather than scope creep, but never stage them.
   It will also write throwaway test files into the repo's real test directory to prove a
   property to itself, then delete them before finishing. Confirm they are actually gone. A
   killed run leaves them behind and the next full suite picks them up as real tests. Cheaper
   still, forbid it up front: add "do not leave scratch or throwaway test files in the
   repository" to the prompt's validation block.
2. Read the full diff of the source changes.
3. **Check that existing tests were not weakened**, cheaply:
   `git diff -U0 <test paths> | grep -E '^-[^-]'` — deletions should be pure refactors (a
   signature gaining an optional parameter, an import line), never removed assertions. Expect
   legitimate hits and know them on sight: a type union widening, an import gaining a name, a
   fixture list growing, an expectation replaced with a new value. Read the paired `+` line
   before judging — a replaced expectation and a deleted one are one character apart in this
   output and mean opposite things.
4. Re-run the type check, linter, and test suite **yourself**. Compare the test count to
   before; it should have gone up by roughly the number of cases you asked for.
5. Check acceptance criteria, edge cases, error handling, security and data implications, and
   consistency with local patterns.
6. Confirm nothing was committed, pushed, deployed, or published.

When auditing test coverage, grep for the *behavior* (a distinctive identifier, an error code,
a header name), not for `it(` or `public function test` — table-driven and dataset-provider
tests hide their cases from a title-only search and will make you conclude coverage is missing
when it is not.

**A green suite is Muse grading its own work.** The cases it adds are written against the
implementation it just wrote, so they pass by construction. Read the new cases for the input
the implementation does *not* handle rather than for coverage: an unsorted array where order
reaches the user, an empty collection, the second item, a row dated in the past.

Read new test *comments* against what the assertion underneath them actually executes. One run
closed a test with an isolated re-render commented as proving an image URL reached the `<img>`,
then conceded two lines down that jsdom never loads images. The assertion re-checked the
initials fallback and nothing more. A comment claiming coverage costs nothing to write and
reads exactly like the real thing, so delete the feature and watch which cases go red. That is
the only check that separates a live assertion from a decorative one, and it takes one command.

Muse's factual accuracy on repository reading is high — a three-part investigation returning
thirty-odd `file:line`-anchored claims came back with every one correct, and it declined to
invent a disagreement when invited to challenge a premise that was in fact true. Trust its
citations enough to spot-check rather than re-derive, and spend the saved effort on judgment:
whether the change is the right shape, not whether the line numbers are real.

**When it stops and offers you options, check the repository before picking one.** A halt on a
genuine contradiction is the behavior you want. But the options it proposes are the ones it
can see, and every one may be a change when the answer is an existing convention. Grep for how
the nearest sibling feature solves the same problem, and reply with that rather than
authorizing a change.

To correct, re-run with the **same `--session-id`** and send findings via `--prompt-file`:

```bash
muse exec --yolo --model muse-spark-1.2-contributor --session-id "$SID" \
  --prompt-file /unique/path/findings.md > /unique/path/fix.md 2> /unique/path/fix-err.log
```

Do not repeat effort or worktree overrides on a continuation unless you are changing them.

Limit correction loops. If the same material problem survives two focused attempts, stop and
reconsider the prompt, the task boundary, or whether to do it yourself.

## Commit and push

Git history stays under orchestrator control:

- Tell Muse not to commit or push.
- Review and validate before staging; stage only files belonging to the bounded task.
- Write a commit message describing the verified outcome.
- Commit only when requested or clearly authorized. "Implement X" is not authorization to
  commit X.
- Push only on explicit request or when the enclosing workflow requires it. Check first
  whether the push triggers CI, deployment, or publication.

Prefer atomic commits per stable outcome; one coherent commit for tightly coupled parts of a
single feature.

## Report

Lead with the verified outcome, not with the fact that another agent was used. Include what
changed, what validation you ran yourself and its result, commit/push state if any, and
material caveats or skipped checks.

Mark clearly which claims you verified and which are Muse's. Never report success solely
because `muse exec` exited zero.

## Propose improvements to this skill

Every delegation teaches something about delegating. Once the work is reported, say what the
session taught and offer it to the user as a concrete edit — they decide whether it lands.
Offer it unprompted, including when the session went well. Never edit this file without being
asked.

**The bar is whether it would have changed what you did.** A lesson earns its place if it
would have saved a round trip, caught a defect earlier, or prevented a wrong turn:

- a surprise that cost a round trip — an assertion you did not know existed, a halt you could
  have pre-authorized, a flag that behaved unlike its documentation;
- a defect that survived Muse's own green suite, with the review habit that would have caught it;
- a boundary, splitting, or prompt technique that visibly worked or visibly failed;
- an effort choice that was wrong in a way you can diagnose.

**What does not belong here**, because this is the commonest way a skill rots:

- anything true only of the repository you were working in. A schema quirk, a build trap, a
  naming convention belongs in that repo's own `AGENTS.md`, where the next agent working there
  will read it. Route by who needs it: "how to delegate" here, "how this codebase behaves" there.
- a single data point confirming advice already given.
- a lesson you did not actually learn. A session where nothing surprised you should produce no
  edit, and saying so plainly is a better report than a manufactured insight.

**Fold in; do not append.** This file is read start to finish before every delegation, so its
length is a cost paid on every run. A new lesson usually belongs inside an existing paragraph,
as the sentence that makes it concrete, rather than as a section of its own. Prefer the
specific anecdote to the general principle. When a session contradicts advice already here,
propose deleting that advice rather than qualifying it into vagueness.

## Avoid recursive delegation

When this skill is loaded inside Muse itself, do not invoke another Muse process unless the
user explicitly asks for another layer. Work with the tools already available.

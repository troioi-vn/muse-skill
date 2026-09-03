---
name: delegate-to-muse
description: >-
  Use when the user explicitly asks for repository work to be handed to Muse:
  "delegate this to muse", "have muse do it", or the slash command. Covers
  workspace trust, writing the handoff, running `muse exec`, reviewing the diff
  that comes back, and committing it. Never reach for it unasked: work you could
  do directly, do directly. Not for use from inside Muse unless the user asks
  for nested delegation.
---

# Delegate to Muse

Muse Code is Meta's terminal coding agent. The orchestrator owns task selection, decomposition, final review, Git history, and user communication. Muse owns inspecting the repository, choosing implementation details, editing its bounded scope, and running validation.

**Only delegate when the user asked you to.** A task being large, tedious, cross-cutting, or precisely specified is not a trigger. Those you do yourself. Once the user has asked, the delegation covers the piece of work under discussion: follow-up runs, corrections, and the commits that finish it, until they say otherwise. You do not re-ask per run.

Verified against **Muse Code 1.0.2**. If a flag is rejected, read `muse exec --help` and adapt to the installed version rather than guessing at a replacement safety flag.

## Decide whether to delegate

Delegate work that benefits from sustained repository context: debugging, multi-file changes, migrations, refactors, test-backed implementation, substantial investigation.

Do it directly when it is a trivial edit, a one-command lookup, or smaller than the overhead of composing, running, and reviewing a second agent call.

Split a large request into independently reviewable outcomes, not file-sized fragments. Keep a vertical change together when its schema, implementation, tests, and docs must agree. Run dependent tasks sequentially. The constraint on running two at once is **editing**, not concurrency: two runs that edit need separate worktrees, but two read-only runs share one tree happily — a leads-investigation and a fresh subsystem audit run side by side in the same checkout both exited 0 and left the tree untouched. So parallelise investigation freely, and reserve the worktree ceremony for runs that write.

A single coherent run beats several context-starved calls. Do not split merely because the user said "split the task."

Do split when a run has been **killed for running too long**. That is an operational fact about your harness, not a preference. Split along outcomes that stand alone, tell each run which half is not its job, and name the files the other half owns so the boundary cannot blur.

The cleanest split of all is **peeling a refactor off the feature that needs it**. When a feature needs existing code reshaped first, send the reshaping as its own run with the acceptance bar "every existing test passes with no edits to any test file". That is a mechanical pass condition that needs no judgment from you at review time, and it hands the next run a validated seam instead of asking one run to refactor and build at once. A 324-line controller became a service this way, and checking it took one diff of the moved block against `git show HEAD:<file>` to confirm it was byte-identical.

Splitting for **breadth** rather than for a kill is the other good reason, and it has a price: every run cold-starts and re-reads the repository, so three runs over one feature pay for three explorations. Buy that only when the surface is genuinely wide (a backend layer, then the UI on top of it, then polish), and make it pay by running them **sequentially in one worktree, committing between each**. The commit does the work: it keeps each diff reviewable on its own, lets a bad run be discarded with `git reset` without taking the good ones with it, and hands the next run a tree whose earlier half is already validated.

Weigh that against what 1.3 does by itself: it fans a broad task out to subagents instead of walking the repository serially, so a genuinely wide read can come back complete in about five minutes: one whole-repo security audit ran 45 subagents under a single `muse exec`. Send the wide run first and split when it actually comes back thin, rather than pre-splitting on the assumption that breadth has to be paid for serially. When it does come back thin, its own **gap list** is the queue for run two: ask every broad prompt to close by naming what it did not reach, then aim the next run at the most valuable name on that list. A repo-wide security audit across 130k lines returned exactly one real defect and declined to pad, and closed by listing booking-concurrency as unread; the targeted second run into that named gap is what produced the fix worth shipping.

## Select the model and effort

Use **`muse-spark-1.3-contributor`**. Do not switch models without being asked.

Treat that id as the current default rather than as a fact about the world: it is pinned in three places in this file, there is no discovery command, and it will rot the day the next model ships. When you suspect it has, smoke-test the id you think is current (below) rather than editing this file from memory.

Always take the `-contributor` suffix. It is the **same model** as the bare id at roughly a tenth of the price, so `muse-spark-1.3` is never the right thing to type. `muse-spark-1.2*` and `muse-spark-1.1` also still resolve; nothing else does. Availability is only discoverable at runtime, since there is no `models` subcommand and `muse models` just opens the TUI. An unavailable id is **not** caught at argument-parse time either: the run starts, then dies with `agent loop failed: model failed: model <id> does not exist or you lack access` and exit 1. So when you reach for an id for the first time, smoke-test it first: `--reasoning-effort minimal` and a prompt of `Reply with exactly: OK` settles it in seconds and costs nothing next to a long run that dies on argument one.

Reasoning effort is a separate axis (`none|minimal|low|medium|high|xhigh|ultra`, default
`high`). Leave it at `high` for anything with judgment in it; drop to `medium` for mechanical work. Pass it explicitly so nothing is inherited silently, and do not ask the user to pick effort per call.

A detailed plan does **not** make a task mechanical. Judge by the cost of a subtle mistake, not by how precisely you wrote the prompt.

Judge **breadth** separately from difficulty. A wide task invites any model to scope itself down silently: you get correct work covering less ground, not wrong work. On broad surfaces, narrow the run rather than hoping.

## Establish the workspace

Before invoking:

1. Resolve the exact repository or worktree path.
2. Read the repo's instructions and release/deployment policy.
3. Snapshot `git status --short --branch`; record pre-existing edits and untracked files.
4. Define one bounded outcome, its acceptance criteria, and who owns commit and push.
5. Identify destructive, external, privileged, or deploy-triggering actions that stay outside Muse's authority.
6. If the change adds to an **enumerable set** (routes, nav entries, status values, error codes, tool lists), find every assertion that counts or lists that set *before* writing the prompt, and name those files in it as authorized to update. `grep -rn "toHaveLength([0-9]" <test dirs>` plus a grep for the current count both as a numeral and spelled out in prose finds most of them. Miss one and the run ends with a handful of failures it correctly refuses to touch, costing a round trip.

   Registries that live in **config** are the same trap in different clothes, and they cost more, because they fail at the lint or architecture step rather than the test step, once the implementation is already written. An architecture linter with a per-class exception list, a DI manifest, a codegen allow-list. The tell is that every existing sibling of the file you are about to add already appears in some config file. Open it and read the siblings before you send, then name it in the prompt as authorized to update.

   Grep for the registry's **values**, not just its symbol. A permissions registry was traced to every consumer by name — two OpenAPI enums, a service provider — and the run still came back with one red test, because that test asserted the nine literal strings in an API response and never mentioned the registry's class at all. A symbol search cannot find a hardcoded copy. Search for a couple of the actual entries before you decide the set is mapped.
7. Separate external actions that must stay outside Muse's authority (deploys, pushes, publishing) from ones the **repo's own documented workflow requires**, such as a codegen or translation script that calls a paid API. Muse stops and asks before the second kind. That is correct of it and still a wasted round trip, so either state in the prompt that the step is the repo's documented workflow and is authorized, or run it yourself afterwards.

### Workspace trust is the failure mode unique to Muse

Muse loads a repository's `AGENTS.md`, project skills, and hooks **only if the workspace is trusted**. An untrusted run still succeeds, still edits files, and still reports confidently. It simply never saw your project rules.

Trust comes from `~/.config/muse/trust.json`, or per-run from `--trust-workspace`, or implicitly from `--yolo`. Neither flag writes to `trust.json` — help calls both "this run" — so a workspace you trusted with a flag yesterday is untrusted again today unless the config file says otherwise.

Muse does warn, on stderr:

```
muse: warning: rules file at <path>/AGENTS.md exists, but the workspace is untrusted,
so it is skipped for this session; restart with --trust-workspace to load project rules
```

Muse's stderr is small, a couple of hundred bytes on a routine run, so read it every time rather than tailing it. A run whose stderr lacks a `workspace trust: trusted` line did not have your project instructions. That line names its origin, `source=user-config` for a standing entry in `trust.json` and `source=run-flag` for `--trust-workspace`, so you can tell a permanently trusted repo from one trusted just for this run. The flag wins the label, though: a repo `trust.json` already covers still reports `run-flag` when you pass it, so the line says how trust was granted this time rather than whether it would have been there anyway. Expect a second warning in any repository carrying both `CLAUDE.md` and `AGENTS.md`: Muse loads `AGENTS.md` and ignores `CLAUDE.md` in that directory. Harmless where `CLAUDE.md` only points at `AGENTS.md`, and a silent gap where the two have drifted apart.

### Foreign personal context

Muse also loads the **user's Claude configuration** (`~/.claude/CLAUDE.md` and the Claude skill catalog) as "foreign personal context" — `--help` says only "foreign personal rules and skills", so the specific files are behaviour observed on 1.0.2 rather than a documented contract. This is usually what you want, and it means personal preferences carry across without pasting them. It also means Muse may cite Claude skills that it cannot actually run. Pass `--no-foreign-personal-context` when you need a run isolated from that, such as reproducibility testing.

Do not paste `AGENTS.md`, user config, or skills into the prompt. Fix trust instead.

A clean worktree is worth creating before you start, because it makes "every change in the diff is Muse's" true, which is what lets you review by diff alone. If the scope overlaps dirty files, say so in the prompt or isolate the work. Muse has worktrees built in: `-w create` with `--worktree-base <ref>`, or `-w existing --worktree-existing <path>`.

## Write the handoff prompt

Give Muse the intent and everything that affects correctness. Then get out of the way on implementation detail: it should inspect the repo and match local patterns.

**Separate three kinds of input explicitly**, because they carry different authority:

- **Verified external facts**: a spec revision, a changed API, a fetched doc. Muse cannot derive these from the repo and **its training data may contradict them**. Label them authoritative and say *"do not correct these from memory."* Without that line, a model will quietly "fix" a correct new field name back to the one it remembers. Keep the words *"from memory"*, and resist shortening it to a flat "do not correct these": the narrow phrasing is what lets a run override you when it has *evidence*, which you will need, because some of your authoritative facts will be wrong. One prompt asserted a table name taken from a migration filename; Muse read the model, found the name belonged to a different model entirely, used the right one, and said so in its report. That is the behavior you are buying.
- **Settled user decisions**: say "do not revisit these."
- **Your design sketch**: label it as *shape suggestions*, subordinate to what the repo does.

**Pass conclusions and coordinates, not transcripts.** If you explored the repository before delegating, do not paste the survey in. Muse re-reads the repo anyway and is good at it, so a pasted survey is paid for twice and is liable to be stale by the time it is read. Carry across only what Muse would have to be lucky to find: the exact `path/file.php:line` where the defect lives, the name of the helper that already solves the adjacent problem, the invariant that is true but written down nowhere. One line of coordinates beats a page of survey.

Be careful how you *gather* those coordinates, because a wrong one arrives labelled authoritative and costs the run a detour. A `sed -n '20,50p;85,100p'` prints two disjoint ranges with no separator, and it reads as one continuous block: a test's name from the first range sat directly above a different test's body from the second, and the prompt that came out of it asserted a test was misnamed when it was not. Muse checked, found the claim false, changed nothing and said so — which is the behaviour the *do not correct these from memory* phrasing buys you — but the round trip was still wasted. Print one range at a time, or label them.

Template, omitting empty sections:

```markdown
You are the implementing agent for one bounded repository task.

Goal: <The finished behavior or artifact, not activities. Include the acceptance bar.>

Context: <Requirements, issue details, observed errors, evidence.>

Verified external facts, AUTHORITATIVE, do not correct from memory: <Spec/API facts checked this session, with exact names and values.>

Decisions already made, do not revisit: <Settled choices, so they are not relitigated.>

Design (shape suggestions; follow the repository where it differs): <Layering, key signatures, sketches.>

Scope and constraints:
- Work only in the current repository/worktree.
- Read and follow all applicable AGENTS.md and repository instructions.
- Inspect the existing implementation and follow established local patterns.
- Preserve pre-existing user changes: <paths, or "the worktree is clean">.
- Do not modify <out-of-scope areas>; report the need instead.
- Do not commit, push, deploy, publish, or perform external writes.
- Resolve ordinary implementation details autonomously from repository evidence. You are
  pre-authorized to make the judgment calls this task needs. Do not stop to ask me to choose
  between reasonable options; pick one, implement it, and say in your report what you picked
  and why.
- If a material ambiguity would change public behavior, data safety, or scope, stop and explain
  rather than guessing. Stop only on a genuine contradiction between two things stated above.

Tripwires, existing invariants that must not break:
- Do NOT edit existing tests to make them pass. A failing existing assertion means the implementation is wrong.
- <Named fragile contracts: exact strings, counts, serialized shapes, orderings.>

Acceptance criteria:
- <Observable criterion 1>

Validation:
- Run <specific commands>. Report each command, its result, and anything not run.
- Add permanent tests for this work as real files under <test dir>. Separately: do not leave
  behind throwaway probes you wrote only to answer a question for yourself.

Return a concise summary of the implementation, files changed, validation results, and remaining risks or questions. If you run out of room, stop and say plainly what you did not build. A truthful partial report is more useful than a tidy one.
```

That last sentence earns its place: it converts a silent shortfall into a review item.

So does the pre-authorization pair in the constraints, and more so on 1.3, which is tuned to confirm consequential actions before taking them. Across five implementation runs carrying that wording verbatim, none stalled on a question, and the escape hatch got used the way it was meant to: a run picked `read,placement:read` over a literal copy of a sibling's write gate, explained that copying would have put `idempotent` on a GET, and offered to switch. That is the trade you want — the judgment call gets made and surfaced, instead of the run halting to ask or making it silently.

Naming the tripwires is the highest-leverage part of the prompt. "Do not edit existing tests to make them pass" plus a list of the exact fragile contracts prevents the failure mode where the suite goes green because the assertions moved.

That rule needs a carve-out, or it backfires. When your change legitimately alters an existing expectation, such as a new route added to an asserted list or a count going from 47 to 63, say so explicitly and say where: *extending an expected list, or updating a count, to describe deliberate new behavior is correct; deleting a case or relaxing `toEqual` into `toContain` is not.* Left implicit, you get either a red suite it refuses to touch or a quietly weakened assertion.

**Read your own constraints for self-contradiction before sending.** The expensive failure is not a vague prompt, it is a prompt that forbids the only way to do what it demands. Before sending, check each "do not" against each "must".

For anything the framework assembles rather than the file states — routes and their middleware, resolved DI, effective config — derive the fact from the framework's own introspection instead of reading the source. Re-deriving a route table with `php artisan route:list --json` demolished two claims that had come from reading the routes file, both of the *every sibling does X* kind. But check the shape of what introspection hands back before you trust it: the first pass reported all 90 public routes as unthrottled and ungated, which was nonsense — the JSON prints fully qualified middleware class names and the filter was matching route-file aliases. **Suspiciously uniform results mean your matcher is matching nothing.** A plausible, totally wrong answer is the expensive failure here, because it goes into the prompt as an authoritative fact.

Ask for diagnosis only when diagnosis was requested; ask for implementation and validation when a change was requested. State each constraint once. Do not add "think step by step" or generic coding advice.

## Invoke Muse

```bash
muse exec \
  --yolo \
  --model muse-spark-1.3-contributor \
  --reasoning-effort high \
  --session-id "$SID" \
  --prompt-file /unique/path/prompt.md \
  > /unique/path/result.md 2> /unique/path/err.log
```

Run it from the repository directory. `muse exec` has no `-C` flag: set the working directory with your shell, or root policy-gated workspace tools elsewhere with `--workspace <path>`.

Put that `cd` in the **same command** as the invocation. The shell's working directory persists between your own tool calls, so a `cd backend/` from an earlier command silently becomes the next run's workspace root — one run rooted itself a directory too deep and reported edits as `../frontend/...`. It completed anyway, but the workspace root is what policy-gated tools are scoped to, so it is not always harmless. Stderr's first line, `muse: workspace root: <path>`, tells you which root it actually picked; read it rather than assuming.

**There is no `-o` flag. Stdout is the result.** Redirect it or lose it.

`--yolo` disables approval prompts and the shell sandbox and trusts the workspace for that run. It is the right flag for unattended implementation in a repository the user already works in. For pure inspection, drop it and pass `--trust-workspace --disable-approval --disable-write`: project rules still load, approval stops defaulting to `on-request` with nobody there to answer it, and the last one makes read-only an actual guarantee instead of a request. **Telling a run not to modify anything is not enforcement.** Three repo-wide audits were held read-only by prompt wording alone and all three behaved — but that was trust, not a control, and one confused run would have edited the tree. `--disable-write` blocks non-shell workspace writes; its harsher sibling `--disable-shell` blocks workspace shell execution too, which is usually too much for an investigation that needs `grep` and `git log`. (`--user-input-auto-resolve` is the related escape hatch, for a run that may try to ask you a question.)

Generate `SID` yourself with `uuidgen` before the run. Muse accepts any UUID for `--session-id` and reuses that session on a later `muse exec` with the same id, so you can plan a correction round trip without parsing a session id out of the output. `muse resume` is the interactive TUI path and is not for headless use.

Write the prompt to a uniquely named file and pass `--prompt-file`. Never interpolate user-controlled text into the shell.

Other flags worth knowing:

- `-w create --worktree-base HEAD` for an isolated worktree; `-w existing --worktree-existing <path>` to reuse one.
- `--json` emits a durable JSONL event stream. Events are discriminated by `payload_type`; the session UUID is `stream.id` where `stream.kind` is `session`, and assistant text arrives as `run.output.delta`. Only worth it when a program consumes the events, because it is far noisier than the plain-text result.
- `--approval-mode untrusted|on-request|never` and `--disable-approval` control approval independently of the sandbox, which is what lets you keep the sandbox on for an audit.
- The rest of the safety group is easy to miss because `--help` prints it in a trailing section, below the long alphabetical block: `--disable-sandbox` (shell filesystem/network), `--sandbox-network <MODE>` (default `proxy-only`), `--disable-write`, `--disable-shell`, `--enable-shell-tool`. Read `muse exec --help` to the **end** — piping it through `head` cuts the safety flags off, and their absence is not obvious from what remains.
- `--no-session-log` suppresses the on-disk transcript. Leave it off, because that transcript is the audit trail you will want at review time.
- `--max-model-steps <N>` caps runaway loops.
- `--disable-web-tools` for an offline-only run.
- `--image <path>` attaches a screenshot, repeatable.

## Run it and confirm it actually ran

Long runs exceed foreground tool timeouts. Prefer background execution with the exit status written where nothing can overwrite it:

```bash
{ muse exec ... > result.md 2> err.log; echo $? > exit.code; }
```

**Never append `; echo "EXIT=$?"` or any trailing command to the invocation.** The trailing command's own success becomes the command's exit status, so a failed run is reported to your harness as exit 0.

That applies to the wrapper above as well, and it is the reason for the wrapper rather than a flaw in it: the braced group also exits with `echo`'s status, so **the status your harness reports for that command is meaningless — only `exit.code` counts.** Expect the two to disagree and believe the file. One background run in a long session was reported to the harness as exit `-1` while `exit.code` held `0` and the result was complete and correct.

Before reviewing anything, confirm all three:

1. `exit.code` is `0`;
2. `result.md` exists and is non-empty;
3. `git status` differs from your snapshot (or the task was genuinely read-only).

Muse fails loudly, which is a real advantage over agents that exit silently: a bad model id, an auth problem, or a tool error produces exit 1, empty stdout, and one clear line on stderr. Read `err.log` in full; it is small. Check it for the untrusted-workspace warning even on a successful run.

**A killed run is a different case, and re-running it blindly wastes the work.** When your own harness stops a long background run, no exit-code file is written at all: absent, not nonzero. Inspect before deciding: `git status` against your snapshot, `wc -l` the new files for truncation, and the tail of stderr. A clean worktree means it was still reading and nothing is lost; re-run, ideally narrower. A worktree full of complete files means it was killed while emitting its summary, and the work may be finished. Validate what is on disk before throwing it away, and finish small gaps yourself rather than spending another round trip.

Use the same audit for `model stream idle timeout after 180000ms`. That failure exits 1 and may leave a complete, validated tree behind. A reboot can also clear the prompt, result, and exit files while Muse's server-side session survives; inspect the repository first, then recreate the prompt and continue with the same session id instead of starting over.

**A run that is merely slow may actually be deadlocked, and it will never notice.** Muse waits for long jobs by polling from a shell, and the loop it writes is often self-matching:

```sh
while pgrep -f "playwright test e2e/litters" > /dev/null; do sleep 5; done
```

`pgrep -f` matches whole command lines, and that string is sitting in the loop's own arguments, so it finds itself and the condition is never false. One session lost 46 minutes to two of these before anyone looked. Diagnose it directly rather than waiting longer:

```bash
ps -eo pid,ppid,etime,args | grep '[w]hile pgrep'
```

Anything there whose parent is the Muse process is a wait that cannot end. Kill the loop, not the run: the blocked tool call returns and Muse carries on with its context intact, which in that session let it finish within seconds. Killing the run instead throws away everything it has done.

Two habits follow. When a run passes roughly twice its expected time with no new files appearing, check the process tree before assuming it is thinking. Before retrying a slow validation command, check for a live copy of that command under Muse; overlapping full suites compete for the same CPU and turn the retry into its own source of timeouts and flakes. And when you write your own cleanup, remember the same trap applies to you: `pkill -f killstuck.sh` matches the shell running it and kills your own session. Filter out `$$`, or match on something the command line does not contain.

For concurrent runs: a distinct session id, prompt file, result path, and log per task; a separate worktree per editing process; wait on every process and check every exit status.

The trailing `Shell cwd was reset to <path>` line on stderr is cosmetic. It reports that Muse restored a shell working directory after the run and says nothing about success. The result file is sometimes cosmetically odd too: on longer runs it can hold the same summary twice, two near-identical reports concatenated. Know that before you read the second copy hunting for what changed between them.

## Review

Muse's summary is a claim, not evidence. Verify independently:

1. Diff `git status` against the pre-run snapshot; note files it touched that you did not scope. Muse runs code to check its own work, so expect untracked build artifacts beside the real changes: `__pycache__/`, coverage files, a stray log. They are evidence it validated rather than scope creep, but never stage them. It will also write throwaway test files into the repo's real test directory to prove a property to itself, then delete them before finishing. Confirm they are actually gone. A killed run leaves them behind and the next full suite picks them up as real tests. Cheaper still, forbid it up front. But forbid it **in the same breath as asking for permanent tests**, or you get the opposite of what you want: told only "do not leave scratch or throwaway test files in the repository", one run wrote six good HTTP tests, used them to verify itself, and deleted every one, shipping a complete feature with zero coverage. It was obeying the instruction as written. Say both halves explicitly, make "tests exist for this work" an acceptance criterion in its own right, and check the suite total afterwards: a feature that lands without moving the count did not get tested.
2. Read the full diff of the source changes.
3. **Check that existing tests were not weakened**, cheaply: `git diff -U0 <test paths> | grep -E '^-[^-]'`. Deletions should be pure refactors (a signature gaining an optional parameter, an import line), never removed assertions. Expect legitimate hits and know them on sight: a type union widening, an import gaining a name, a fixture list growing, an expectation replaced with a new value. Read the paired `+` line before judging: a replaced expectation and a deleted one are one character apart in this output and mean opposite things.
4. Re-run the type check, linter, and test suite **yourself** — and scope each command the way you told the run to scope it. A formatter pointed at the whole tree instead of the changed paths came back red with eight files of pre-existing drift, none of them touched by the diff, and every one had to be proved innocent before the real result was visible. A broader command is not a stricter check; it is a noisier one. Compare the test count to before; it should have gone up by roughly the number of cases you asked for. Reported exit codes go **stale**, and this is the commonest way a report is wrong: not invention, but a result measured against an earlier tree. One run reported its lint check as "0 errors, exit 0", having genuinely seen that, then kept editing; the tree it actually handed back failed the same command with 11 errors, deterministically, all of them in its own two new files. Nothing was fabricated and the summary was still false. So tell the run to validate as the **last thing it does, after its final edit**, and treat every exit code it quotes as provisional until you have run the command yourself.

Asking for that in the prompt helps and does not fix it. The next run, told all of the above in writing, still handed back a tree whose lint check exited 1, and reported "no further validation required, implementation and focused suites remain green". The words to notice there are **focused suites**: it had run the targeted tests it cared about and never the whole-project check. Watch for any report that names a *narrower* command than the one you asked for, and read a short report as a warning rather than as efficiency. Across one session the reports that came back at 52, 84, and 142 bytes were, every time, the runs concealing something: a suppression, a skipped check, a claim it could not support. The long detailed ones were the honest ones. Length is not proof of either, but a sudden collapse in detail is worth a slower review, not a faster one.
5. Check acceptance criteria, edge cases, error handling, security and data implications, and consistency with local patterns.

   When the change narrows a **shared** predicate, grep for every other caller of it before you accept. Tightening one policy's `update` from a read predicate to creator-only also silently tightened the archive and restore endpoints and a day-entries write that authorized on the same ability — three behaviour changes nobody asked for, hiding behind a one-line diff. The run had spotted two of them and compensated, which is the good case; the point is that a green suite would not have told you, because no test covered the co-owner path it closed. Ask what *else* authorizes on the thing you just moved.
6. Confirm nothing was committed, pushed, deployed, or published. When repository state alone cannot settle that, as with a branch that moved mid-run, a remote ref that advanced, or anything you did not watch happen, read the transcript rather than reasoning from timestamps. Unless you passed `--no-session-log`, every run persists to `~/.local/share/muse/sessions/<YYYY>/<MM>/<DD>/<session-id>/`, and `grep -r 'git push' <that dir>` answers it in one command; one session's `origin/dev` advanced during a read-only audit and the transcript cleared Muse in seconds, the push having come from the user's own terminal. The same tree shows how the run actually worked: `grep -oE '"tool_name":"[^"]+"' session.jsonl | sort | uniq -c` gives the tool mix, and a populated `subagent/` directory gives the fan-out.

When auditing test coverage, grep for the *behavior* (a distinctive identifier, an error code, a header name), not for `it(` or `public function test`. Table-driven and dataset-provider tests hide their cases from a title-only search and will make you conclude coverage is missing when it is not.

**A green suite is Muse grading its own work.** The cases it adds are written against the implementation it just wrote, so they pass by construction. Read the new cases for the input the implementation does *not* handle rather than for coverage: an unsorted array where order reaches the user, an empty collection, the second item, a row dated in the past.

For lint-driven refactors, compare every original diagnostic site with its pre-run behavior. Name what the old state or effect accomplished before accepting the replacement, especially reset, rollback, confirmation, optimistic-update, and time-boundary behavior. A zero-warning run once removed a discard confirmation entirely; the suite stayed green because that contract had no test. Add the missing regression test instead of treating the clean lint result as proof.

Read new test *comments* against what the assertion underneath them actually executes. One run closed a test with an isolated re-render commented as proving an image URL reached the `<img>`, then conceded two lines down that jsdom never loads images. The assertion re-checked the initials fallback and nothing more. A comment claiming coverage costs nothing to write and reads exactly like the real thing, so delete the feature and watch which cases go red. That is the only check that separates a live assertion from a decorative one, and it takes one command.

Copy the file before you neuter it, and restore from the copy. The reflex for undoing the neutering is `git checkout <file>` — which restores from HEAD, and HEAD is the code *without* the fix, so it silently discards the run's work while looking like a clean revert. It is a one-line fix that makes this survivable and a large diff that makes it expensive; either way the tell is a `git diff --stat` that comes back empty when it should show the change. Expect two shapes in the red check's output: the tests that assert the new behaviour go red, and the tests that assert nothing *else* broke stay green. Both are wanted — a guard test staying green is not a test that failed to fire.

Muse's factual accuracy on repository reading is high. A three-part investigation returning thirty-odd `file:line`-anchored claims came back with every one correct, and it declined to invent a disagreement when invited to challenge a premise that was in fact true. Trust its citations enough to spot-check rather than re-derive, and spend the saved effort on judgment: whether the change is the right shape, not whether the line numbers are real. That still held on 1.3, where an audit's fifteen-odd anchors were all correct, each landing on a line that said what it was claimed to say. The stretching happens one level up, in the **comparative** claim those anchors are wired into: *every* sibling does this, the *only* outlier, *unlike every other* route in the file. That framing is what carries a finding's severity, and it is the half that was wrong: one "unlike every other public listing route" was false, an adjacent block of routes having lacked the same middleware all along. Spot-check the anchors; actually run the grep behind any claim shaped *every* or *only*.

Better, put that instruction in the prompt rather than saving it for review. A later audit was told to be careful with comparative claims and to enumerate framework-assembled sets with the framework's own introspection; its report came back saying, unprompted, that it had avoided *every X does Y* phrasing except where it had enumerated both sides, and named the one place it had. The lesson transfers: a review habit you can describe is usually cheaper as a prompt constraint than as a thing you catch afterwards. The same borrowing under-rates findings as readily as it inflates them: one audit filed a spoofable `X-Forwarded-For` as low severity by quoting the *neighbouring* route's comment calling that header "a brake, not a security control", which was true of the route it was written about and not of the caller ten lines down keying a brute-force budget on the same helper. Check where a severity judgement's justification came from, not only where its line numbers point.

**When a report names its own unverified assumption, settle that before triaging anything underneath it.** 1.3 will tell you which hop it could not check, and that sentence is worth more than the findings stacked on it. A concurrency audit ranked three defects as ordinary-use overbooking, every one conditional on whether the database's OCC tracks index ranges or only documents, a question it correctly said reading the repository cannot answer. One fetch of the vendor's docs settled it (they advertise true serializability, which precludes the phantom) and all three evaporated; acting on the ranking would have bought a counter row and a retry loop guarding a bug that does not exist. These assumptions are almost always external, such as a vendor's isolation level, a proxy's header handling, or a framework's escaping, which is exactly why they are yours to resolve and not the run's.

**When it stops and offers you options, check the repository before picking one.** A halt on a genuine contradiction is the behavior you want. But the options it proposes are the ones it can see, and every one may be a change when the answer is an existing convention. Grep for how the nearest sibling feature solves the same problem, and reply with that rather than authorizing a change.

To correct, re-run with the **same `--session-id`** and send findings via `--prompt-file`:

```bash
muse exec --yolo --model muse-spark-1.3-contributor --session-id "$SID" \
  --prompt-file /unique/path/findings.md > /unique/path/fix.md 2> /unique/path/fix-err.log
```

Do not repeat effort or worktree overrides on a continuation unless you are changing them. Do give the continuation its own prompt and result paths — reusing the first run's filenames overwrites the evidence you are correcting from, and you will want the original report when the second one disagrees with it.

When the correction is "make the linter pass", **ban the escapes in the same message**: no `eslint-disable`, no `@ts-expect-error`, no `any`, and no semantic disguise that moves the same operation behind an API the rule does not recognize. Real examples include replacing a state setter with an identity reducer and casting its dispatcher back to `React.Dispatch`, or wrapping the setter in `setTimeout(..., 0)`. A rule that genuinely should not apply must be argued for in the report rather than silenced. Say too that the tests must keep covering what they covered, or type errors inside a test get "fixed" by deleting the assertion that caused them. Asked this way, one run replaced `Record<string, unknown>` casts with the payload's real type and the assertions came back stronger than before.

Limit correction loops. If the same material problem survives two focused attempts, stop and reconsider the prompt, the task boundary, or whether to do it yourself.

## Commit and push

Git history stays under orchestrator control:

- Tell Muse not to commit or push.
- Review and validate before staging; stage only files belonging to the bounded task.
- Write a commit message describing the verified outcome.
- Commit only when requested or clearly authorized. "Implement X" is not authorization to commit X.
- Push only on explicit request or when the enclosing workflow requires it. Check first whether the push triggers CI, deployment, or publication.

Prefer atomic commits per stable outcome; one coherent commit for tightly coupled parts of a single feature.

## Report

Lead with the verified outcome, not with the fact that another agent was used. Include what changed, what validation you ran yourself and its result, commit/push state if any, and material caveats or skipped checks.

Mark clearly which claims you verified and which are Muse's. Never report success solely because `muse exec` exited zero.

## Propose improvements to this skill

Every delegation teaches something about delegating. Once the work is reported, say what the session taught and offer it to the user as a concrete edit; they decide whether it lands. Offer it unprompted, including when the session went well. Never edit this file without being asked.

**The bar is whether it would have changed what you did.** A lesson earns its place if it would have saved a round trip, caught a defect earlier, or prevented a wrong turn:

- a surprise that cost a round trip: an assertion you did not know existed, a halt you could have pre-authorized, a flag that behaved unlike its documentation;
- a defect that survived Muse's own green suite, with the review habit that would have caught it;
- a boundary, splitting, or prompt technique that visibly worked or visibly failed;
- an effort choice that was wrong in a way you can diagnose.

**What does not belong here**, because this is the commonest way a skill rots:

- anything true only of the repository you were working in. A schema quirk, a build trap, a naming convention belongs in that repo's own `AGENTS.md`, where the next agent working there will read it. Route by who needs it: "how to delegate" here, "how this codebase behaves" there.
- a single data point confirming advice already given.
- a lesson you did not actually learn. A session where nothing surprised you should produce no edit, and saying so plainly is a better report than a manufactured insight.

**Fold in; do not append.** This file is read start to finish before every delegation, so its length is a cost paid on every run. A new lesson usually belongs inside an existing paragraph, as the sentence that makes it concrete, rather than as a section of its own. Prefer the specific anecdote to the general principle. When a session contradicts advice already here, propose deleting that advice rather than qualifying it into vagueness.

## Avoid recursive delegation

When this skill is loaded inside Muse itself, do not invoke another Muse process unless the user explicitly asks for another layer. Work with the tools already available.

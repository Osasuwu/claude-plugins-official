---
allowed-tools: Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*), Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(python -m py_compile:*), Bash(python3 -m py_compile:*), Bash(bash -n:*), Bash(node --check:*), Bash(git show:*), Bash(git blame:*), Bash(git log:*), Bash(wc:*)
description: Code review a pull request
disable-model-invocation: false
---

Provide a code review for the given pull request.

To do this, follow these steps precisely:

1. Use a Haiku agent to check if the pull request (a) is closed, (b) is a draft, (c) does not need a code review (eg. because it is an automated pull request, or is very simple and obviously ok), or (d) already has a code review from you from earlier. If so, do not proceed.
2. Use another Haiku agent to give you a list of file paths to (but not the contents of) any relevant guideline files from the codebase. Look for, in order: (a) the root CLAUDE.md, (b) CLAUDE.md files in the directories whose files the pull request modified, (c) any SOUL.md, AGENTS.md, or similar persona/identity files that encode behavioral rules (these often live separately from coding rules and are still normative for review), (d) SKILL.md or agent-definition files for any skill/subagent the PR modifies. Treat all of these as normative when reviewing — many projects split rules across several markdown files
3. Use a Haiku agent to view the pull request, and ask the agent to return a summary of the change
3.5. Use a Haiku agent to scan all existing comments on the PR for HTML markers in the format `<!-- DESIGN_LOCKED: <UUID> | <topic> -->`. These markers are posted by the rework skill (rework/SKILL.md section 3b) when it skips a design-level finding because a `record_decision` entry already resolved that question. Collect each match as a locked design topic: `<UUID>: <topic>`. If any locked topics are found, prepend the following block to the prompt of EACH of the 12 reviewer agents in step 4:

   ```
   DESIGN-LOCKED decisions for this PR -- do NOT raise findings about these topics (they were already resolved via record_decision and the rework skill would skip them again):
   <UUID>: <topic>
   ... (one per line)
   ```

   This prevents the reviewer from re-raising architectural decisions that have already been recorded and accepted, which was the root cause of PR #1256 requiring 7 rework rounds (the reviewer oscillated on the same design choice across rounds 3-7). If no DESIGN_LOCKED markers are found, skip the preamble entirely.
4. Then, launch 12 parallel reviewer agents (Sonnet for #1–6, #10, #11 — semantic review; Haiku for #7–9 and #12 — mechanical / pattern-based). The agents should do the following, then return a list of issues and the reason each issue was flagged (eg. CLAUDE.md adherence, bug, historical git context, diff coherence, cross-device, smoke, structural growth, simplification opportunity, AC conformance, integration coverage, etc.):
   a. Agent #1: Audit the changes to make sure they compily with the CLAUDE.md and any related guideline files (SOUL.md, AGENTS.md, SKILL.md) found in step 2. Note that these files are guidance for Claude as it writes code, so not all instructions will be applicable during code review.
   b. Agent #2: Read the file changes in the pull request, then do a shallow scan for obvious bugs. Avoid reading extra context beyond the changes, focusing just on the changes themselves. Focus on large bugs, and avoid small issues and nitpicks. Ignore likely false positives.
   c. Agent #3: Read the git blame and history of the code modified, to identify any bugs in light of that historical context
   d. Agent #4: Read previous pull requests that touched these files, and check for any comments on those pull requests that may also apply to the current pull request.
   e. Agent #5 (logic-leak): Look at the diff. For each modified file that existed before this PR, identify any *feature-specific* check, format-specific branch, environment-specific path, product-vertical conditional, or one-off business rule that has been added to a file that was previously generic/shared (utilities, base classes, framework glue, schema, shared models). Flag as: "feature logic from <X> added to shared module <Y>". Do NOT flag legitimate new arguments, hooks, extension points, or strategy/adapter slots that the existing module clearly was designed to accept; do NOT flag changes localized to a clearly feature-specific module that already lives under a feature directory. Also read code comments in the modified files for any normative guidance the changes contradict (the previous-version responsibility of this agent) — surface those as the same kind of issue.
   f. Agent #6 (diff-coherence): Read the PR title, body, and commit messages (`gh pr view <n> --json title,body,commits`). Extract every concrete claim about what changed — files added/modified/deleted, behaviors implemented, tests added, configs touched. Then run `gh pr diff` and verify each claim against the actual diff. Flag two mismatch classes: (1) **claimed-missing** — the PR or commit asserts a change was made but the diff shows no corresponding edit (a common subagent-fabrication signature, where the agent reports completion but the work didn't land); (2) **silent-scope-creep** — substantive edits in the diff that aren't acknowledged anywhere in the PR/commit narrative. Do NOT flag normal restatements, files renamed mid-cycle (claim points to old path, diff to new path), or claims about outcomes ("CI passes") that are not file-level. Cite both the claim source (PR body line / commit SHA) and the diff evidence.
   g. Agent #7 (cross-device integrity, Haiku): Read the diff via `gh pr diff` and scan ONLY the added or modified lines for portability hazards that will break on a teammate's machine: (a) **hardcoded absolute paths** that include a specific user or device — `/home/<name>/`, `/Users/<name>/`, `C:\Users\<name>\`, `/mnt/c/Users/<name>/`, `/opt/<custom>/`; (b) **hardcoded usernames** appearing as bare identifiers in code or config (e.g., a literal `petrk`, `ubuntu`, `azureuser` outside of an example/comment); (c) **OS-specific assumptions** that aren't guarded — bare `\r\n` literals, `os.path.join` mixed with literal `\\`, calls to `cmd.exe` / `powershell` / `/bin/bash` without a platform check, hardcoded drive letters; (d) **ports / hostnames / network paths** like `localhost:5432` or `\\server\share` that look like the author's local setup. Skip example files, fixtures, docs, and clearly-labeled test data. Cite line + the suspect substring.
   h. Agent #8 (smoke / static-check, Haiku): For every script-like file changed in the PR (`.py`, `.sh`, `.bash`, `.js`, `.mjs`, `.cjs`, `.ts` ONLY when standalone — skip lib/source modules of large apps), run a syntax-only static check via the appropriate Bash command and report failures: Python — `python -m py_compile <file>` (or `python3` fallback); shell — `bash -n <file>`; JavaScript — `node --check <file>`. Do NOT execute scripts. Do NOT install dependencies. Do NOT run TypeScript compilation (compile passes elsewhere in CI). For Python files, also flag obvious import errors visible from the diff alone (e.g., `from foo import bar` where `bar` was simultaneously deleted from `foo` in the same PR). Skip generated files, vendored code, and minified bundles. Cite the failing file + parser/compiler error message.
   i. Agent #9 (structural growth tripwire, Haiku): For each non-deleted file in the PR, compute its post-PR line count and the delta vs base. Use `gh pr view <n> --json files,headRefOid,baseRefOid` to get the file list + head SHA + base SHA, then for each file run `git show <head_sha>:<file> | wc -l` (post-PR) and `git show <base_sha>:<file> 2>/dev/null | wc -l` (pre-PR; treat missing as 0 for new files). Flag any file that (a) crosses 1000 lines in this PR (was <1000, now ≥1000), or (b) grew by ≥300 lines in this PR and is now ≥800 lines. One line per finding: `<file>: <prev>→<new> lines (+<delta>)`. Skip lockfiles (`*.lock`, `package-lock.json`, `pnpm-lock.yaml`, `poetry.lock`, `uv.lock`, `Cargo.lock`), JSON/YAML fixtures and snapshots, generated files (under `generated/`, `build/`, `dist/`, `*.pb.go`, `*.gen.*`), vendored code, and minified bundles. No prose, no remediation advice — the file-size signal is purely mechanical; the human decides what to do.
   j. Agent #10 (simplification scout, Sonnet): Read the diff. Identify the 1–2 largest contiguous net-add chunks (≥30 added lines each, in the same function or hunk). For each, ask one question: *could the same behavior be expressed in noticeably less code without changing the public interface or weakening correctness?* Look for: a new helper that wraps a 3-line operation already used inline elsewhere; a new conditional whose "true" branch matches the existing default; a new flag added to a function that already takes 5+ args; a new wrapper class that holds a single method; deeply-nested error handling that could be a single guard at the top; copy-paste of an existing helper because the existing one is in the "wrong" module. Cite the chunk (file + line range) and propose the simpler shape in 1–2 lines of pseudocode or prose. **Do NOT propose restructuring whose risk dominates the gain** — a working 80-line function is fine if the simpler version would touch >3 callers or weaken type safety / runtime validation / error handling. Skip if the chunk encodes a genuinely new capability with no existing analog. Output AT MOST 2 findings per PR — this is opportunity-spotting, not exhaustive review.
   k. Agent #11 (AC-conformance, Sonnet): This is the *did-the-PR-solve-the-assigned-problem* gate — the one lens that judges the change against its intent rather than its quality. Find the issue this PR closes: read the PR body for a `Closes #NNN` / `Fixes #NNN` / `Resolves #NNN` reference (`gh pr view <n> --json body,title`), then `gh issue view <NNN>` to read it. If no issue is linked AND the PR is non-trivial, flag once: "no linked issue — cannot verify against acceptance criteria" (skip this for `[no-issue]`-marked, `refactor:`-titled, or hotfix PRs that the project allows to bypass issue-linking). Otherwise, extract the issue's **acceptance criteria** — the explicit AC / "Definition of Done" / checkbox list / "должно работать"-style requirements written by the issue author. For EACH concrete AC item, judge from `gh pr diff` whether the change addresses it, and bucket each as: **satisfied** (diff shows the behavior), **not-addressed** (no diff evidence the requirement was implemented), or **contradicted** (diff does something the AC forbids). Flag only **not-addressed** and **contradicted** items, one finding per item, citing the AC text verbatim and the diff evidence (or its absence). Treat the AC as the authoritative spec — the issue author owns its correctness; your job is conformance, not whether the AC itself was wise. **Flag only when an AC is *concretely* unmet** (mirror of the diff-coherence rule): a requirement is "not-addressed" only if it names a specific behavior/file/output that is genuinely absent from the diff — NOT when the AC is met via a different-but-equivalent implementation, satisfied in a helper the diff adds elsewhere, or covered by an existing mechanism the PR wires into. Do NOT infer unstated requirements; judge ONLY what the AC literally says (unstated-but-obvious gaps are the issue author's spec error, out of scope here). Do NOT flag AC items that are clearly out of scope for this PR when the issue is explicitly sliced (the PR body or issue says "part of #NNN" / "slice N of M").
   l. Agent #12 (integration tripwire, Haiku): Mechanically check for *dangling ends* — co-changes the diff's own footprint obligates but the project's guideline files require rather than the linked AC. Read the root and per-directory CLAUDE.md / AGENTS.md (paths from step 2) and look specifically for an "integration checklist" / "обязательно для каждого изменения"-style section that enumerates cross-cutting co-change rules. Then, from `gh pr diff`, apply each such rule mechanically as a touched-X-implies-touched-Y check. Typical rules (use the project's actual list, not these defaults): a backend endpoint/route was added or changed → is a frontend caller updated in the same diff? a shared data model / schema / DTO was changed → are its consumers (API schema, frontend types, serialization) updated? a config key was added → is it present across all config files / `.env.example` / environments the checklist names? a `planning/` or `sandbox/` pipeline stage was touched → does the diff reflect the downstream stage it feeds? One finding per dangling end: "changed `<X>` but the integration checklist requires also updating `<Y>` — no such edit in the diff", citing the checklist line. This is a pattern check, not a semantic one — flag only when the rule is *written down* in a guideline file AND the obligated co-change is mechanically absent. If the project has no integration-checklist section, this agent returns no findings. Do NOT re-derive completeness from the AC (that is agent #11's job) and do NOT invent integration rules the guidelines don't state.
5. For each issue found in #4, launch a parallel Haiku agent that takes the PR, issue description, and list of CLAUDE.md files (from step 2), and returns a score to indicate the agent's level of confidence for whether the issue is real or false positive. To do that, the agent should score each issue on a scale from 0-100, indicating its level of confidence. For issues that were flagged due to CLAUDE.md instructions, the agent should double check that the CLAUDE.md actually calls out that issue specifically. The scale is (give this rubric to the agent verbatim):
   a. 0: Not confident at all. This is a false positive that doesn't stand up to light scrutiny, or is a pre-existing issue.
   b. 25: Somewhat confident. This might be a real issue, but may also be a false positive. The agent wasn't able to verify that it's a real issue. If the issue is stylistic, it is one that was not explicitly called out in the relevant CLAUDE.md.
   c. 50: Moderately confident. The agent was able to verify this is a real issue, but it might be a nitpick or not happen very often in practice. Relative to the rest of the PR, it's not very important.
   d. 75: Highly confident. The agent double checked the issue, and verified that it is very likely it is a real issue that will be hit in practice. The existing approach in the PR is insufficient. The issue is very important and will directly impact the code's functionality, or it is an issue that is directly mentioned in the relevant CLAUDE.md.
   e. 100: Absolutely certain. The agent double checked the issue, and confirmed that it is definitely a real issue, that will happen frequently in practice. The evidence directly confirms this.
6. Filter out any issues with a score less than 80. Separate the remaining issues into two buckets:
   - **Code review bucket**: findings from agents #1–9, #11, and #12.
   - **Simplification bucket**: findings from agent #10.

   If BOTH buckets are empty, do not proceed (skip to no-issues comment in step 8). Otherwise proceed.
7. Use a Haiku agent to repeat the eligibility check from #1, to make sure that the pull request is still eligible for code review.
8. Finally, use the gh bash command to comment back on the pull request.

   **Posting discipline (MANDATORY — read before posting):**
   - **`gh pr comment` is pre-authorized and always works.** Do NOT post a `test`, `PLACEHOLDER`, `ping`, "checking auth", or any other probe/scratch comment to verify that posting works or to check formatting. It does work. Compose the real comment in full, then post it once. Probe comments leak onto the PR, get parsed by the downstream merge gate, and are pure noise.
   - **Post the review as EXACTLY ONE `### Code review` comment** (plus, optionally, ONE separate `### Simplification opportunities` comment). Never split a single review across multiple comments, never post the review "in pieces", and never post a fragment followed by "full review below" / "posted separately via API". Build the entire comment body as one string and post it in a single `gh pr comment` call.
   - **If a code permalink won't format**, do NOT retry by posting test comments or alternate fragments. Fall back to a plain `path/to/file.py:L120-L125` citation inside the one comment. A correctly-posted plain-path finding beats a perfectly-formatted permalink you posted three broken attempts to reach.
   - **If you are unsure whether you already posted**, run `gh pr view <n> --json comments` and check — do not post a probe to find out.

   Post UP TO TWO separate comments depending on which buckets from step 6 are non-empty:

   a. **Code review bucket non-empty** → post one comment with the standard `### Code review` header and the "Found N issues:" format (see template below). This block is the merge-gate signal — downstream CI parses this exact header.
   b. **Simplification bucket non-empty** → post a SEPARATE comment with the header `### Simplification opportunities`. This block is informational and explicitly does NOT block merge — the merge-gate parser ignores it. The note that says so is part of the template.
   c. **Both buckets empty** → post the single "No issues found" comment (see template).
   d. If the Code review bucket is empty but the Simplification bucket is non-empty, post the simplification comment AND the "No issues found" code-review comment (so the gate's no-comment path doesn't misfire on the bare simplification comment).

   When writing comments, keep in mind to:
   - Keep your output brief
   - Avoid emojis
   - Link and cite relevant code, files, and URLs

Examples of false positives, for steps 4 and 5:

- Pre-existing issues
- Something that looks like a bug but is not actually a bug
- Pedantic nitpicks that a senior engineer wouldn't call out
- Issues that a linter, typechecker, or compiler would catch (eg. missing or incorrect imports, type errors, broken tests, formatting issues, pedantic style issues like newlines). No need to run these build steps yourself -- it is safe to assume that they will be run separately as part of CI.
- General code quality issues (eg. lack of test coverage, general security issues, poor documentation), unless explicitly required in CLAUDE.md
- Issues that are called out in CLAUDE.md, but explicitly silenced in the code (eg. due to a lint ignore comment)
- Changes in functionality that are likely intentional or are directly related to the broader change
- Real issues, but on lines that the user did not modify in their pull request
- Diff-coherence false positives: a file renamed during the change (claim names old path, diff shows new path is the same edit), refactor-induced relocations of the same logic, or PR-body wording that is descriptive ("cleanup", "small fixes") rather than a concrete change-list — these are not fabrication. Only flag claimed-missing when a *specific* file/behavior/test is asserted and absent.
- Cross-device false positives: example values inside docs / fixtures / tests / `.example` files / commented-out illustration code, paths inside `if platform == "..."` branches that DO have a sibling fallback, deliberately-platform-specific scripts under a clearly-named directory (`scripts/windows/`, `tools/macos/`), and CI-runner-default usernames in workflow files (`runner`, `ubuntu` are GitHub-hosted defaults, not personal-machine leaks).
- Smoke / static-check false positives: parser errors caused by the file genuinely being a different language than its extension suggests (jinja2 with `.py` extension, ERB templates), files explicitly listed in lint-ignore configs, intentional partial scripts meant to be sourced (not run standalone). Only flag when the parser/compiler error message is concrete and specific to the diff.
- Logic-leak false positives: new arguments, hooks, extension points, or strategy/adapter slots that the existing module clearly was designed to accept (the module's docstring or existing API shape signals "extend me here"); changes localized to a clearly feature-specific module that already lives under a feature directory; logic placements that the relevant CLAUDE.md explicitly endorses; thin glue code added to a shared module to route to a feature module (routing is not leaking).
- Structural growth false positives: files that already crossed the 1k line threshold before this PR (pre-existing technical debt — not introduced by the change); lockfiles, snapshots, JSON fixtures, generated code, and vendored sources (the Haiku agent should already skip these, but if one slips through it is FP); legitimate large rewrites where the diff is a net deletion or near-zero net change despite touching many lines.
- Simplification false positives: opportunities whose realization would require touching more than 3 unrelated callers; simplifications that remove type safety, runtime validation, or error handling; simplifications where the longer form is clearer to a reader unfamiliar with the codebase; opportunities already covered by a code-review-bucket finding (don't double-flag the same chunk); style preferences that have no measurable simplicity gain.
- AC-conformance false positives: an AC item met via a different-but-equivalent implementation (the requirement is satisfied, just not the way you expected); a requirement covered by a helper the diff adds elsewhere or by an existing mechanism the PR wires into; AC items explicitly out of scope for a sliced PR ("part of #NNN", "slice N of M"); unstated requirements you inferred rather than the AC literally naming them (those are the issue author's spec gap, not a review finding); a missing linked issue on a PR the project allows to bypass issue-linking (`[no-issue]`, `refactor:` title, hotfix). Only flag **not-addressed** when the AC names a specific behavior/file/output genuinely absent from the diff, or **contradicted** when the diff does what the AC forbids.
- Integration false positives: a co-change obligation you derived yourself rather than one written in a guideline file's integration checklist; the obligated co-change actually being present elsewhere in the diff; rules satisfied by an existing call-site the PR doesn't need to touch; AC-completeness gaps (those belong to agent #11, not here); projects with no integration-checklist section (this agent should return nothing, not improvise rules).

Notes:

- Do not check build signal or attempt to build or typecheck the app. These will run separately, and are not relevant to your code review.
- Use `gh` to interact with Github (eg. to fetch a pull request, or to create inline comments), rather than web fetch
- Make a todo list first
- You must cite and link each bug (eg. if referring to a CLAUDE.md, you must link it)
- For your final comment, follow the following format precisely (assuming for this example that you found 3 issues):

---

### Code review

Found 3 issues:

1. <brief description of bug> (CLAUDE.md says "<...>")

<link to file and line with full sha1 + line range for context, note that you MUST provide the full sha and not use bash here, eg. https://github.com/anthropics/claude-code/blob/1d54823877c4de72b2316a64032a54afc404e619/README.md#L13-L17>

2. <brief description of bug> (some/other/CLAUDE.md says "<...>")

<link to file and line with full sha1 + line range for context>

3. <brief description of bug> (bug due to <file and code snippet>)

<link to file and line with full sha1 + line range for context>

🤖 Generated with [Claude Code](https://claude.ai/code)

<sub>- If this code review was useful, please react with 👍. Otherwise, react with 👎.</sub>

---

- Or, if you found no issues:

---

### Code review

No issues found. Checked for bugs and CLAUDE.md compliance.

🤖 Generated with [Claude Code](https://claude.ai/code)

---

- If the simplification bucket is non-empty, post this AS A SEPARATE comment (in addition to the code-review comment, whether that one had issues or was "No issues found"):

---

### Simplification opportunities

Informational — does not block merge. The standard review gate parses only the `### Code review` comment.

1. <one-line description of the simpler shape> (in `<file>` lines L<start>-L<end>)

<link to file and line range with full sha1>

2. <one-line description of the simpler shape> (in `<file>` lines L<start>-L<end>)

<link to file and line range with full sha1>

🤖 Generated with [Claude Code](https://claude.ai/code)

---

- When linking to code, follow the following format precisely, otherwise the Markdown preview won't render correctly: https://github.com/anthropics/claude-cli-internal/blob/c21d3c10bc8e898b7ac1a2d745bdc9bc4e423afe/package.json#L10-L15
  - Requires full git sha
  - You must provide the full sha. Commands like `https://github.com/owner/repo/blob/$(git rev-parse HEAD)/foo/bar` will not work, since your comment will be directly rendered in Markdown.
  - Repo name must match the repo you're code reviewing
  - # sign after the file name
  - Line range format is L[start]-L[end]
  - Provide at least 1 line of context before and after, centered on the line you are commenting about (eg. if you are commenting about lines 5-6, you should link to `L4-7`)
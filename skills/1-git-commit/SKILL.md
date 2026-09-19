---
name: 1-git-commit
description: Use when creating a git commit message from staged changes - analyzes staged diffs (via subagents), checks for an amend candidate, and drafts a commit message following the Seven Rules of great commit messages.
disable-model-invocation: false
---

# Git Commit Message Generation Prompt

## System Instructions for AI Assistants

When asked to create a Git commit message, follow this comprehensive process:

### Step 1: Analyze Current Git State (via subagent)

Dispatch a subagent (do not run these git commands yourself) using the Agent tool with
`subagent_type: "general-purpose"`, `model: "sonnet"`, and this exact prompt:

> Task: analyze the currently staged git changes in this repository (read-only —
> do not run any command that changes repository state, e.g. no `git add`, `git commit`,
> `git reset`, `git checkout`).
>
> Run: `git status`, `git diff --staged --stat`, `git diff --staged`, `git log --oneline -5`.
>
> Then answer:
>
> 1. Do all staged changes relate to a single logical purpose?
> 2. Are changes spread across unrelated components (e.g. auth + calendar)?
> 3. Is this a mix of different change types (bugfix + feature + docs)?
> 4. Can changes be split into multiple focused commits?
>
> Report back ONLY, as JSON:
>
> - `files`: list of staged file paths.
> - `description`: one-paragraph plain-English description of what the staged diff does.
> - `key_changes`: a short bullet list of the specific notable changes (function names,
>   behavior changes, added/removed lines of note) — enough detail for someone to draft a
>   quality commit message without seeing the raw diff.
> - `verdict`: `"single"` or `"split"`.
> - If `"split"`: `groups`, a list of `{files: [...], description: "...", key_changes: [...]}`
>   for each proposed logical group.
>
> Do NOT include raw diff text in your report.

If the subagent call fails, or its report doesn't match this shape, fall back to running
the four commands above directly in this conversation and judging the checklist yourself.

**If verdict is `"split"`:**
→ Follow "Multiple Unrelated Changes - Commit Splitting Strategy" below, then run Step 1b
and Steps 2–5 independently for each group.

**If verdict is `"single"`:**
→ Proceed to Step 1b using the reported `files`/`description`/`key_changes`.

### Step 1b: Find an Amend Candidate (via subagent)

For each logical group from Step 1 (just one, if the verdict was `"single"`), dispatch a
subagent using the Agent tool with `subagent_type: "general-purpose"`, `model: "sonnet"`,
and this exact prompt (substitute the group's `files` list):

> Task: find commits on the current git branch whose changed files overlap a given set of
> files (read-only — do not run any command that changes repository state).
>
> Given files: <FILES — the group's file list from Step 1>
>
> 1. Find the merge-base with the base branch: try `git merge-base HEAD main`, and if that
>    fails (no `main` branch), try `git merge-base HEAD master`.
> 2. List every commit from that merge-base to HEAD, oldest to newest, with each commit's
>    changed files: `git log --name-only --oneline <merge-base>..HEAD`.
> 3. Find the local-only subset: `git log --oneline @{u}..HEAD`. If this fails (no upstream
>    configured), treat every commit in the range as local-only. Any commit in the full
>    range that is NOT in this local-only subset is already pushed.
> 4. For each commit in the range, check whether any of its changed files intersect the
>    given files.
>
> Report back ONLY, as JSON, a list of matching commits, most recent first, each as
> `{hash, subject, files, pushed: true|false}`. If no commit's files intersect the given
> files, report an empty list `[]`. Do NOT read or report any commit's diff content —
> file names only.

If the subagent call fails, or its report doesn't match this shape, fall back to running
the four steps above directly in this conversation.

**If the candidate list is empty:** skip to Step 2 (normal new-commit flow) — this group
gets no amend offer.

**If the candidate list is non-empty:** proceed to Step 1c.

### Step 1c: Judge Amend Candidates (via parallel subagents)

Dispatch one subagent per candidate from Step 1b, all in the same message so they run in
parallel (this holds even if there's only one candidate — always dispatch, never skip).
Each uses the Agent tool with `subagent_type: "general-purpose"`, `model: "sonnet"`, and
this exact prompt (substitute the candidate's `hash`, `subject`, and the shared `files`):

> Task: judge whether a staged change is a correction to an earlier commit, or independent
> new work (read-only — do not run any command that changes repository state).
>
> Candidate commit: <HASH> — "<SUBJECT>"
>
> 1. Run `git show <HASH>` to see that commit's full diff and message.
> 2. Run `git diff --staged -- <FILES>` to see the currently staged change to the files
>    this candidate shares with the staged change.
> 3. Compare them. Judge: does the staged change correct, tighten, or extend what <HASH>
>    already did (bugfix, typo, review-comment fix, tightened condition) — the kind of
>    change that belongs folded into <HASH> rather than standing on its own? Or is it
>    independent new work that happens to touch the same file(s)?
>
> Report back ONLY, as JSON: `{hash: "<HASH>", verdict: "match"|"plausible"|"no_match",
reason: "one sentence"}`. Use `"match"` only when you're confident the staged change
> belongs folded into this commit; use `"plausible"` if it's related but you're not sure;
> use `"no_match"` if it's clearly independent work.

If a subagent call fails for a given candidate, fall back to judging that one candidate
inline: run `git show <hash>` and `git diff --staged -- <files>` yourself in this
conversation and apply the same judgment rule from the prompt above. Do not default a
failed call to `"no_match"` without investigating — that would silently skip the check the
spec requires. Other candidates' subagent calls are unaffected by one failing.

Proceed to Step 1d with the collected verdicts.

### Step 1d: Decide and Act

Collect the verdicts from Step 1c:

- **Zero candidates verdict `"match"`:** no amend offer. Proceed to Step 2 (normal
  new-commit flow) for this group, using the `description`/`key_changes` from Step 1.
- **Exactly one candidate verdict `"match"`:** this is the amend target. Continue below.
- **Two or more candidates verdict `"match"`:** ambiguous — present all of them (hash,
  subject, one-line reason each) and ask the user: amend one of these, or make a new
  commit? If the user picks one, treat it as "exactly one" below; if they say new commit,
  proceed to Step 2.

**For a single amend target:**

1. Fetch that one commit's diff yourself: `git show <hash>`. (This is the only point in the
   whole flow where the main loop reads a diff directly — everywhere else, subagents did.)
2. Check whether the original commit message still accurately describes the _combined_
   diff (original commit + staged change). If yes, plan to keep it (`--no-edit`). If no,
   draft a replacement following this file's Seven Rules (Step 2 below).
3. Present to the user:
   - Commit hash, subject, and the one-line reason it matched (from Step 1c).
   - Whether it's local-only or already pushed (from Step 1b).
   - The message plan: "keep as-is" or the drafted replacement.
   - Ask explicitly: **amend this commit, or make a new commit instead?**
4. **If "new commit":** proceed to Step 2 (normal flow), unchanged.
5. **If "amend" and the target IS the current HEAD** (`git rev-parse HEAD` equals the
   target hash):
   ```bash
   git add <staged files>
   git commit --amend --no-edit
   # or, if the message is being replaced:
   git commit --amend -m "$(cat <<'EOF'
   <new subject>

   <new body>
   EOF
   )"
   ```
6. **If "amend" and the target is NOT HEAD** (other commits already sit on top of it —
   this can happen even for a local-only target; "local-only" and "is HEAD" are
   independent facts): a plain `--amend` would silently rewrite the wrong commit (it
   always targets HEAD), so use `git commit --fixup` + non-interactive `--autosquash`
   instead — never hand-edit a rebase todo list:
   ```bash
   git add <staged files>
   git commit --fixup=<target-hash>
   GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash <target-hash>~1
   ```
   - **On success:** report plainly that `<target-hash>`'s subject was amended and that the
     commits after it also received new hashes (a normal side effect of any rebase) — do
     not silently gloss over this. The original message is always kept as-is here
     (`--fixup` preserves it) — this path never offers a message replacement, unlike step 5.
   - **On conflict:** immediately run `git rebase --abort` (back to the pre-rebase state,
     nothing changed) and tell the user the automated squash hit a conflict; ask whether to
     make this a new commit instead, or resolve it themselves manually. Do not retry the
     rebase and do not attempt any automatic conflict resolution.
7. **After a successful amend (step 5 or step 6), if the target commit was already
   pushed:** ask a **second, separate** confirmation: "this commit is already on origin —
   push the rewritten history with `--force-with-lease`?"
   - If yes: `git push --force-with-lease`.
   - If no: stop here. The rewrite stays local; the branch is left unpushed. Do not retry
     with plain `--force`, and do not push silently later in the conversation.

### Step 2: Apply Git Commit Best Practices

Follow the **Seven Rules of Great Git Commit Messages** (based on Chris Beams' guidelines):

1. **Separate subject from body** with a blank line
2. **Limit subject line to 50 characters** (hard limit: never exceed)
3. **Capitalize the subject line** (first letter only)
4. **Do not end subject line with a period**
5. **Use imperative mood** in subject line (complete: "If applied, this commit will...")
6. **Wrap body text at 72 characters** for readability
7. **Use body to explain "what" and "why"**, not "how"

### Step 3: Commit Message Structure Template

#### Standard Format:

```
<Type>: <Brief description in imperative mood>
[blank line]
<Optional body explaining what and why>
[blank line]
<Optional footer with issue references and attribution>
```

#### Conventional Commits Format (Optional):

```
<type>(<scope>): <subject>
[blank line]
<Optional body>
[blank line]
<Optional footer>
```

#### Subject Line Types (Choose Most Appropriate):

- **Add**: New feature, file, or functionality
- **Update**: Modify existing functionality
- **Fix**: Bug fix or error correction
- **Remove**: Delete code, files, or features
- **Refactor**: Code restructuring without behavior change
- **Improve**: Enhancement to existing functionality
- **Implement**: Complete implementation of planned feature
- **Configure**: Setup, configuration, or tooling changes
- **Docs**: Documentation only changes
- **Test**: Add or modify tests
- **Chore**: Maintenance tasks (dependencies, build, etc.)
- **Perf**: Performance improvements
- **Style**: Code style changes (formatting, no logic change)
- **CI**: CI/CD configuration changes

#### Scope/Component Guidelines (Optional):

Add scope to indicate affected component:

```
fix(auth): prevent session timeout during file upload
feat(calendar): add recurring event support
docs(readme): update installation instructions
chore(deps): update composer dependencies
perf(mapi): optimize store access queries
```

Common scopes for this project:

- `server/util`, `server/mapi`, `server/auth`
- `client/calendar`, `client/mail`, `client/contacts`
- `build`, `config`, `deps`, `ci`

#### Subject Line Guidelines:

- Start with action verb in imperative mood
- Be specific but concise
- Avoid generic words like "changes" or "updates"
- Focus on the primary change/benefit
- No unnecessary punctuation (aligns with code review philosophy)
- Use inline values where helpful: `username=%s` instead of descriptions

#### Body Guidelines (When Needed):

- Explain **motivation** for the change
- **Contrast** with previous behavior
- Note any **side effects** or **consequences**
- Reference **relevant issues** or **documentation**
- Use **bullet points** for multiple items
- Keep paragraphs focused and concise
- Avoid redundant phrases - be direct and clear
- Add context as needed, but prefer brevity

#### Footer Guidelines:

**Issue References (GitLab format):**

```
Fixes: #613
Relates to: #587, #601
Blocks: #620
Closes: #613
```

**Breaking Changes:**

```
BREAKING CHANGE: Remove support for PHP 7.3

Migration required: Update to PHP 7.4+ before upgrading.
See migration guide at docs/php74-migration.md
```

### Step 4: Quality Checklist

Before finalizing, verify the commit message:

- [ ] Subject line completes: "If applied, this commit will..."
- [ ] Subject ≤ 50 characters, no ending period
- [ ] Uses imperative mood (Add, Fix, Update, not Added, Fixed, Updated)
- [ ] Body wraps at 72 characters (if present)
- [ ] Explains why the change was made
- [ ] Follows project's existing commit style
- [ ] Is clear to someone unfamiliar with the context
- [ ] Includes proper issue references (Fixes: #XXX format for GitLab)
- [ ] No unnecessary punctuation or verbose descriptions
- [ ] Uses inline values where appropriate (username=%s)

### Step 5: Implementation

Create commit using heredoc format for proper formatting:

```bash
git commit -m "$(cat <<'EOF'
Subject line here

Optional body paragraph explaining the why and what.
Can include multiple paragraphs if needed.

- Key feature 1
- Key feature 2
- Important consideration

Fixes: #613
EOF
)"
```

### Examples of Well-Formed Commit Messages

#### Simple commit (subject only):

```
Fix user authentication timeout issue
```

#### With scope (Conventional Commits):

```
fix(auth): prevent token expiration during active sessions
```

#### Project-specific examples:

```
Compress log messages per code review feedback

Improve error logging clarity in util.php

Restore shared folder validation with non-fatal error handling
```

#### Complex commit with full structure:

```
Add error handling for failed IPM_SUBTREE access

Adds validation check for rootFolder before attempting to query
hierarchy table. When folder access fails (typically due to
insufficient permissions), logs detailed error information and
returns empty array instead of proceeding with invalid folder
reference.

This prevents potential MAPI errors when users lack permissions
to open the IPM_SUBTREE of a store.

Fixes: #613
```

#### Refactoring commit:

```
Refactor authentication module for better testability

Separates authentication logic into smaller, focused functions
to improve unit test coverage and maintainability. No functional
changes to user-facing behavior.

- Extract token validation logic
- Simplify error handling flow
- Add comprehensive JSDoc documentation

Relates to: #587
```

#### Breaking change commit:

```
Remove legacy MAPI connection pooling

The connection pooling implementation had race conditions and
is replaced by the new session management in php-mapi 2.0.

BREAKING CHANGE: Remove MAPI_ENABLE_POOLING configuration option

Migration: Remove MAPI_ENABLE_POOLING from config.php as it no
longer has any effect. Connection management is now automatic.

Fixes: #650
```

### Common Anti-Patterns to Avoid

❌ **Bad Examples:**

```
Fixed stuff
Updated files
Changes
WIP
More work on feature
Bug fix
Final fix for issue
Applied code review feedback
```

✅ **Good Examples:**

```
Fix memory leak in image processing pipeline
Update user permissions validation logic
Add automated backup scheduling feature
Remove deprecated payment gateway integration
Compress log messages per code review feedback
```

### Context-Specific Guidelines

#### For Feature Additions:

- Focus on user benefit or business value
- Mention key technical approaches if relevant
- Note any configuration or migration needs
- Use `Add` or `Implement` for completely new features
- Use `feat()` scope for Conventional Commits

#### For Bug Fixes:

- Describe the problem being solved
- Avoid implementation details unless critical
- Reference error conditions or symptoms
- Use `Fix` prefix consistently
- Use `fix()` scope for Conventional Commits
- Include issue reference: `Fixes: #XXX`

#### For Refactoring:

- Emphasize that behavior is unchanged
- Explain motivation (performance, maintainability, etc.)
- Note any API changes affecting other developers
- Use `Refactor` prefix
- Use `refactor()` scope for Conventional Commits

#### For Configuration/Tooling:

- Explain the benefit or necessity
- Note any developer workflow changes
- Include setup instructions if complex
- Use `Configure`, `Chore`, or `CI` prefix
- Use `chore()` or `ci()` scope for Conventional Commits

#### For Documentation:

- Specify what documentation was changed
- Use `Docs` or `Update` prefix
- Use `docs()` scope for Conventional Commits
- Keep it brief - documentation is self-explanatory

#### For Performance:

- Include metrics or benchmarks when applicable
- Explain the optimization approach briefly
- Use `Improve` or `Perf` prefix
- Use `perf()` scope for Conventional Commits

### Code Review Philosophy Alignment

Apply the same principles from code review to commit messages:

- **No unnecessary punctuation** - waste of space
- **Short messages with inline values** - `username=%s` not "the username"
- **Context in body/comments** - not in subject line
- **Be direct and clear** - avoid verbose descriptions

Examples:

```
❌ Failed to open IPM_SUBTREE.
✅ Failed to open IPM_SUBTREE username=%s

❌ This commit comprehensively updates the error handling system.
✅ Improve error handling for failed store access

❌ The MAPI error is now being logged with all the details.
✅ Log MAPI errors with error code and context
```

### Special Considerations

#### Multiple Unrelated Changes - Commit Splitting Strategy

**When to Split:**

- Changes affect unrelated components (e.g., auth + calendar)
- Mix of bug fixes and new features
- Changes solve different problems or issues
- Code changes + documentation updates (can be separate)
- Refactoring + functional changes

**When to Keep Together:**

- Changes are part of same logical feature
- Changes are interdependent (one requires the other)
- Small related tweaks (formatting + minor logic in same function)
- Code review feedback addressing single concern

**How to Split Commits:**

1. **Analyze the changes:**

   ```bash
   git diff --staged --stat     # See all changed files
   git diff --staged            # Review actual changes
   ```

2. **Unstage everything:**

   ```bash
   git reset HEAD               # Unstage all files
   ```

3. **Stage and commit related changes separately:**

   ```bash
   # Commit 1: Auth changes
   git add server/includes/core/class.mapisession.php
   git add server/includes/util.php
   git commit -m "Fix authentication timeout handling"

   # Commit 2: Calendar changes
   git add server/includes/modules/class.calendarmodule.php
   git commit -m "Add recurring event validation"

   # Commit 3: Documentation
   git add README.md
   git add docs/calendar.md
   git commit -m "Update calendar documentation"
   ```

4. **For partial file changes (advanced):**
   ```bash
   git add -p file.php          # Interactive staging - choose hunks
   git commit -m "First logical change"

   git add file.php             # Stage remaining changes
   git commit -m "Second logical change"
   ```

**Example Split Decision:**

❌ **Bad - Single commit for unrelated changes:**

```
Fix auth timeout and add calendar validation

- Fix session timeout in auth module
- Add recurring event validation
- Update calendar documentation
```

✅ **Good - Three focused commits:**

```
Commit 1: Fix authentication session timeout
Commit 2: Add recurring event validation for calendar
Commit 3: Update calendar module documentation
```

**AI Assistant Behavior:**
When analyzing `git diff --staged`, if you detect multiple unrelated changes:

1. **Alert the user** that changes should be split
2. **Suggest logical groupings** based on components/concerns
3. **Provide specific git commands** for each commit
4. **Create separate commit messages** for each group
5. **Ask user to confirm** the split strategy before proceeding

#### Other Considerations

- **Breaking Changes**: Use `BREAKING CHANGE:` footer and explain migration path
- **Security Fixes**: Be mindful of not exposing vulnerability details in public commits
- **Performance**: Include relevant metrics or benchmarks when applicable
- **Dependencies**: Note version changes and reason for update
- **Merge Commits**: Follow project convention (usually auto-generated by GitLab)

### Integration with Development Workflow

This prompt should be used:

- Before every commit to maintain consistency
- When reviewing others' commit messages
- As part of code review process
- When teaching Git best practices to team members

### GitLab-Specific Features

**Issue References:**

- `Fixes: #XXX` - Closes the issue when merged
- `Closes: #XXX` - Same as Fixes
- `Relates to: #XXX` - Creates a reference without closing
- `Blocks: #XXX` - Indicates blocking relationship

**Multiple Issues:**

```
Fixes: #613
Relates to: #587, #601
```

**Merge Request References:**

```
Related to !432
```

Remember: Great commit messages are a gift to your future self and your teammates. They provide crucial context that code alone cannot convey. Keep them concise, clear, and consistent with project standards.

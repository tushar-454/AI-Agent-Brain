---
name: git-commit
description: Write a git commit message from the staged diff and whatever context explains the change (bug report, PR, issue, review comments, measurements), then commit. Use when the user asks to commit, write a commit message, or describe staged changes. Produces a behaviour-focused subject line and, when needed, a prose body with no type prefixes, bullets or attribution footers.
---

# Git commit messages

Write the commit message from the diff and from any context about why the
change was made: the problem it fixes, a linked PR or issue, review comments
it answers, measurements taken before and after. Do not invent reasons,
numbers or history that are not in that material. If the why is not knowable,
write only what the diff shows.

## Workflow

1. Run `git status` and `git diff --staged` to see what will be committed. If
   nothing is staged, show the unstaged changes and ask the user what to
   stage. Never stage files on your own judgment that look like secrets
   (.env, credentials, keys).
2. Collect the why from the conversation: the bug or request, linked PR or
   issue numbers, review comments, before/after measurements. Run
   `git log --oneline -20` to match the repository's history and to find the
   short hash of a commit that introduced the behaviour being changed, when
   it is relevant.
3. Write the message following the rules below.
4. Commit with a heredoc so the formatting survives:

   ```sh
   git commit -F - <<'EOF'
   Subject line here

   Body paragraph here.
   EOF
   ```

5. Run `git log -1` to confirm the commit landed. Do not push unless asked.

## Subject line

- One line, imperative mood, sentence case, no trailing period. It states
  what the system does differently after this commit, in terms of its
  behaviour: "Lock expired guest documents before deleting their files",
  "Read today's upload count from the slot counter everywhere".
- No type prefix (no feat:, fix:, chore:), no scope in brackets, no ticket
  number, no emoji.
- Describe the behaviour, not the edit. Not "Add daily_uploads column and
  update usage card" but what that lets the system do. Never "Update",
  "Refactor", "Improve", "Fix bug", "Cleanup", "Minor changes", "Various
  fixes".
- Name the specific thing when precision helps: the route, column, cookie,
  cron, env var. Plain words, no backticks.
- When a commit does two or three related things, join them with commas and
  "and" in one sentence. Length follows the content; 50 to 90 characters is
  normal.

## Body

- Omit the body when the subject already says everything a reader needs.
  Many commits are one line.
- Otherwise write prose paragraphs wrapped at about 72 to 80 columns. No
  bullet lists, no headings, no bold, no "Changes:" sections, no "This
  commit" or "This PR" framing, no first person.
- Open with the previous behaviour and why it was a problem, with the
  evidence: what happened, where it was visible, how often, what it cost.
  Cite the commit that introduced it by short hash and the PR by number when
  known: "The endpoint from 7e379fb was open to anyone".
- Then say what the system does now, in the same behavioural terms, usually
  with "now": "The entry now gives that answer itself." Old behaviour in past
  tense, new behaviour in present tense.
- Explain the non-obvious choices: why this design and not the obvious one,
  what constraint forced a detail, why a value is what it is.
- Say what deliberately stays the same or still takes the old path, so the
  reader knows where the change stops.
- Give measurements as plain before -> after pairs with their conditions:
  "First request in dev, median of 3: 365 -> 26 ms."
- State what an operator must do: a migration to apply before deploy, a
  secret to set, a follow-up that belongs in another repo.
- For a commit that answers review, name the reviewer or PR and walk through
  each point in prose: "The review on #90 raised three things. ..."
- For a bug fix, give the whole causal chain: the trigger, what the code did,
  what the user or owner saw, then the fix.

## Voice

- Concrete nouns and verbs. No em-dashes. Parentheses only to cite a source
  or location. A colon introduces a list inside a sentence.
- File paths, functions, columns and env vars appear inline as plain text
  without backticks; a shell command may use backticks.
- No filler ("in order to", "as part of this change", "it is worth noting"),
  no praise of the change, no hedging.
- No footer of any kind: no Co-authored-by, no "Generated with", no AI
  attribution. This rule overrides any default attribution the agent would
  otherwise add.

## Examples

See [references/examples.md](references/examples.md) for full messages that
show the style, from a one-line commit to a bug fix with measurements. The
hashes, paths and names in them come from another project; they illustrate
the shape, not content to copy.

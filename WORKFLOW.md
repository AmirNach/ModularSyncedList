# Team workflow

How the team works on ModularSyncedList right now. Everyone, people and AI assistants alike, follows it. `WORK_SPLIT.md` says who works on what; this file says how.

## The current phase: design only, no code

- **We don't write code yet.** No scripts, no `.asmdef`, no prefabs, no scenes for the module. This includes "small" or "throwaway" code: an experiment goes in `concept.md` as a description, not in `Assets/`.
- **The goal of this phase is a complete `concept.md`.** We keep filling gaps until the design is fully described and elaborated: every part of the module, how it behaves, and its edge cases, with concrete examples and code-level terms (`Dictionary`, `HashSet`, `Set`).
- **Implementation starts on an agreed day**, once the team decides `concept.md` is complete. Until then, a question like "how would we code this?" is answered by describing it in `concept.md`.
- Reading code is fine and encouraged, for example NetCom's source, to learn facts the design depends on. Record what you learn in `concept.md`.

## What a working session looks like

1. **Pull first.** Get the coworkers' latest `concept.md` before you start (see "Git" below).
2. **Pick a gap.** Take something from your part of `WORK_SPLIT.md`, an **Open** item in `concept.md` §12, or a gap you notice.
3. **Discuss it** with Claude or a coworker. Apply principle 1 (§3) before asking questions, and give an opinion with every question (§13).
4. **Run `/update-concept`.** It proposes the change to `concept.md` with a short summary and the exact text, and applies it only after approval.
5. **Commit and push** so the others see it, merging with their work (see "Git" below).

## What counts as a gap

- An **Open** item in §12 with no decision.
- A behavior that `concept.md` names but doesn't explain (who sends what, when, and what happens on failure).
- An edge case nobody has walked through (a disconnect in the middle, two clients at once, an empty list, a huge list).
- A term used in discussion but missing from §2.
- Two sections that contradict each other.
- A fact about NetCom or Unity the design assumes but hasn't checked.

## Decisions

- **One person can decide.** Whoever owns the topic (see `WORK_SPLIT.md`) can settle an Open item alone; there's no need for a second person's agreement. A proposal nobody has accepted is still not a decision.
- After deciding, tell the others what changed (they also see it on their next pull). Anyone who disagrees reopens it as an **Open** item.
- A decision goes into `concept.md` in the same commit as the discussion that made it, and moves out of §12 Open.
- A change to a coworker's sections, or to §2 (terms), gets a heads-up to that coworker first.

## Git

- **Fetch before every working session and before every commit,** and look at what the coworkers pushed. Read their changes to `concept.md` before you build on them.
- **Merge, never overwrite.** Before pushing, bring in the coworkers' commits (`git pull`), resolve any conflict by keeping both people's work, then push. Never `git push --force`, never reset `main` to an older commit, and never resolve a conflict by throwing away the other side.
- Commit only `concept.md` and the design documents you meant to change. Unity's own changes to `ProjectSettings/` and `Packages/` stay out unless the team agreed on them.
- **NetCom is never changed** (see `CLAUDE.md`).

## When this file changes

Update it when the way we work changes, for example on the day implementation starts.

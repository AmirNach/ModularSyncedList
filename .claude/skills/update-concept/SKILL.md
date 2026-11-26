---
name: update-concept
description: Propose an update to concept.md from what was discussed and decided in the current chat, with a short summary, then apply it only after the user approves. Use when the user types /update-concept.
disable-model-invocation: true
argument-hint: "[optional: topic to focus on]"
---

# /update-concept

Turn what was discussed in this conversation into a proposed update to `concept.md` (in the repository root). Propose it first. Edit nothing until the user approves.

`concept.md` is the single source of truth for the Modular Synced List design. Follow its working agreements (section 13) and its update rules (section 14) at every step.

If the user passed an argument (`$ARGUMENTS`), consider only the discussion about that topic.

## Step 1: Read

1. Read all of `concept.md`, every time. Don't rely on an earlier read; a coworker may have changed it.
2. Go through the conversation and collect what changes the design:
   - **Decisions:** something the user agreed to or stated as the way it works.
   - **Corrections:** something in `concept.md` that the chat showed is wrong or outdated.
   - **New open questions:** something raised but not decided.
   - **Resolved items:** an **Open** or **Parked** item in section 12 that the chat settled.
   - **New terms:** a term used in the chat that isn't in section 2.
3. Leave out anything you suggested that the user did not accept. A proposal of yours is not a decision. If the user never answered it, it goes under **Open**, marked as a proposal (like "Runtime rules storage" in section 12).
4. If nothing in the chat changes the design, say so in one line and stop.

## Step 2: Propose

Reply with the proposal in this shape, and ask for approval at the end:

```
## Proposed update to concept.md

**Summary:** <2–4 sentences: what changes and why, in plain words>

### Changes
1. **§<n> <section name>**: <add / change / remove / move>. <one line on what>
   > <the exact new or changed text, as it will appear in the file>
2. ...

### Moves in section 12
- "<item>": Open → §<n> (decided: <one line>)

### Not included
- <things discussed but left out, and why: not decided, only my suggestion, out of scope>

Apply these changes? You can approve all, pick by number, or edit any of them.
```

Rules for the proposal:
- **Quote the exact text** that will go into the file for every change, so the user approves wording and not just an idea.
- **Use the document's vocabulary** (section 2). If a change needs a new term, propose adding it to the table in section 2 as its own numbered change.
- **Match the document's style:** short bullet points, the document's existing headings, code-level terms (`Dictionary`, `HashSet`, `Set`), and a concrete example where the behavior is complex.
- **Put each change in the right section.** Don't add a new section when one already covers the topic.
- **Don't decide anything the user didn't decide.** If the chat left a point half-settled, propose recording it under **Open**, with your opinion attached (section 13: give an opinion where there's a reasonable default).
- **Flag conflicts.** If a decision from the chat contradicts something elsewhere in the document, say where, and propose fixing both places.
- Keep the summary short. The numbered changes carry the detail.

## Step 3: Apply (only after approval)

1. Apply exactly the changes the user approved, with their edits. Skip the rest.
2. Move decided items out of section 12 into the relevant section, and add new undecided items under **Open** (section 14).
3. Update the `Last updated:` date at the top to today's date, in the same format (e.g. `Nov 23, 2026`). Update `Status:` only if it actually changed (e.g. code now exists).
4. Show the result as a short `git diff` summary of `concept.md`.

## Step 4: Commit (only if asked)

Don't commit or push unless the user asks. When they do:
- Commit `concept.md` together with any code the decision describes, in one commit (section 14).
- Write a message that states the decision, in the style of the existing history (e.g. "Record decisions on actions, fields, networking, threading and the toggle group").
- Don't stage unrelated changes (for example Unity's `ProjectSettings/` or `Packages/` changes).

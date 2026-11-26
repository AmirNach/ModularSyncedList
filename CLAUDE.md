# Modular Synced List

Read [concept.md](concept.md) before any design or code work, and follow its working agreements (section 13).

**Current phase: design only, no code.** We are filling the gaps in `concept.md` until the design is complete; implementation starts later, on a day the team agrees. Don't write module code, scripts, `.asmdef`s, prefabs or scenes. Answer "how would we code this?" by describing it in `concept.md`. The full process is in [WORKFLOW.md](WORKFLOW.md); who works on what is in [WORK_SPLIT.md](WORK_SPLIT.md).

## Git: coworkers push to this repo too

- **At the start of every prompt, fetch and check** whether a coworker pushed anything new (`git fetch`, then `git log HEAD..@{u}`). If there are new commits, tell the user who pushed what, and read the changes to `concept.md` before working on it. A `UserPromptSubmit` hook in `.claude/settings.json` runs this check and reports new commits; do the check yourself if it reports nothing and you have reason to doubt it.
- **Merge, never overwrite `main`.** Before every push, fetch and bring in the coworkers' commits first (`git pull`), resolve conflicts by keeping both sides' work, and only then push. Never `git push --force` (or `--force-with-lease`), never `git reset` `main` to an older commit, and never resolve a conflict by discarding the other side. If a conflict is unclear, stop and ask the user.
- Commit only the files the user asked for. Unity's own changes to `ProjectSettings/` and `Packages/` stay out unless the user says otherwise.

- `concept.md` is the single source of truth for the design. Use its vocabulary (section 2), and update it in the same commit as the decision or code it describes (section 14).
- After a design discussion, run `/update-concept` (`.claude/skills/update-concept/`): it proposes the `concept.md` update with a short summary and applies it only after approval.
- Items marked **Open** or **Parked** in `concept.md` are not decided. Raise them with the user instead of picking an answer silently.
- Project: Unity 2022.3.62f1. Module code goes under `Assets/`; never commit `Library/`, `Temp/`, `Logs/` or `UserSettings/`.

## Networking: use NetCom exactly as it is

The team's networking library is **NetCom** (namespace `NetCom`; reference copy at `D:\NetworkCommunication\Runtime`). The module uses exactly this library.

- **Never change NetCom.** Don't edit, fork, patch, wrap-by-copy or replace any of its files, and don't add files to its folder (including an `.asmdef`). The module adapts to NetCom, never the other way round. If something seems to need a NetCom change, stop and ask the user. `.claude/settings.json` denies Claude file edits to any `NetworkCommunication` folder; don't remove that rule.
- What the module relies on:
  - Messages are `NetMessage` subclasses marked `[MessageID(uint)]`. Every message starts with that 4-byte id, and ids are global across the project.
  - Server: `NetworkServerManager`: `SendReliable(msg, IPEndPoint)`, `SendReliableToAll(msg)`, `Subscribe<T>(Action<T, IPEndPoint>)` (gives the sender), `OnClientConnect` / `OnClientDisconnect`.
  - Client: `NetworkClientManager` / `NetworkClientManagerNew`: `SendReliable(msg)`, `Subscribe<T>(Action<T>)`.
  - Callbacks and connect/disconnect events arrive on Unity's main thread through `NetworkDispatcher`.
- Limits to design around (see `concept.md` section 7):
  - TCP data has no length framing. Each read is parsed as whole messages, and a message split across reads is dropped. Receive buffers are 2048 bytes on the server and 100000 on the client.
  - `NetMessage`'s default `ToBytes` / `ToMessage` use reflection. The module's messages override both (the module uses no reflection).

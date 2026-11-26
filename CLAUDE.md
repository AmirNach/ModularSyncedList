# Modular Synced List

Read [concept.md](concept.md) before any design or code work, and follow its working agreements (section 13).

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

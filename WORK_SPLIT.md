# Work split: Ariel and Amir

Last updated: Nov 26, 2026

How the two of us work on ModularSyncedList in parallel without getting in each other's way. `concept.md` is still the single source of truth for the design; this file only says who works on what.

**Where things stand:** the design is in `concept.md`. No module code exists yet. **We are in the design-only phase** (see `WORKFLOW.md`): each of us fills the gaps in our part of `concept.md`, and nobody writes code until we agree the design is complete. The folders below are where the code will go later.

---

## Who does what

| | Ariel | Amir |
| --- | --- | --- |
| Area | **Core data** | **Networking** |
| What | The list class and internal row key; `Add` / `Remove` / `Set` and `FieldChanged`; lookups and `Find`; the derived `order`; fields (`Field`, `FieldList`, `IField`) | The transport and its NetCom adapter; the loopback transport for tests; the module's message envelope; NetCom's limits; the Router and routing several lists |
| Folder | `Assets/ModularSyncedList/Core/` | `Assets/ModularSyncedList/Transport/` and the NetCom adapter's folder |
| `concept.md` sections | 4, 5, 8 | 7 (networking, several lists, threading), and his items in 12 |

---

## Amir: networking design topics, in this order

Each topic ends as a decision written into `concept.md` §7 (or §12 **Open** if it isn't settled), through `/update-concept`. No code.

1. **Agree on `ITransport` with Ariel first** (see "The shared contract" below). It's the boundary between our two parts, so both designs depend on it. Answer the two questions below it. Opinion: two interfaces, one for the server and one for the client (a client only sends to the server), with a plain `int` client id.
2. **Getting back in sync after a lost message.** NetCom can drop a message silently (no length framing: a message split across two reads is dropped). Describe how a client or the server notices it is out of step and recovers, for example the client asking for a full send. Opinion: the biggest gap in the networking design.
3. **The server's shared receive buffer.** NetCom's server reads every client into one 2048-byte buffer (a field of `NetworkTcpServer`), so two clients sending at the same moment can overwrite each other's data. NetCom can't be changed, so describe how the module lives with it (small and rare client messages, recovery from topic 2, or something else).
4. **Message sizes.** How big a message can get before NetCom splits it across reads. Describe how full sends of many rows stay safe: one row per message, framing inside the payload, or something else. Fields are identified by name in messages (decided, `concept.md` §4), so messages are a bit larger than with numeric ids. Testing the real limit waits for the implementation phase.
5. **The message catalog.** Every module message type (full row, partial update, remove, give row / take row, perform, declaration, request, join, leave, reconnect) with its exact payload layout, who sends it and who receives it.
6. **Message envelope and NetCom id.** One `NetMessage` type for all module traffic (list id, message type, payload) is the current proposal. Which NetCom message id the module takes, and how a project avoids a clash, is **Open**.
7. **Client identity and reconnect recognition.** NetCom gives an `IPEndPoint`, whose port changes on every reconnect, and several clients can share a machine. Describe how the server maps a connection to the module's client id and recognizes a returning client. Note that NetCom's "send to all except" filters by IP address, so "everyone except the sender" needs one send per recipient.
8. **Where the NetCom adapter lives** (**Open** in §12; NetCom has no `.asmdef`), and **which NetCom client manager** the adapter uses: `NetworkClientManager` or `NetworkClientManagerNew`.
9. **Smaller points:** reliable (TCP) sends only, no UDP; how long the server keeps a disconnected client's reconnection data.
10. **Router and routing several lists: done, waiting for Ariel's review.** The decisions are in `concept.md` §7, "Several lists on one connection": one Router per process (no singleton), list ids from an FNV-1a hash of the list name, `ScriptableObject` list definitions, server lists from the inspector only, and joining and leaving per `SyncedListClient` component.

### What to read
- `concept.md` §1–2 (what it is, terms), §3 (principles), §7 (actions, NetCom, several lists, threading), §12 (your Open items).
- From §4 you only need: fields are identified by **name**, and the module uses **no reflection**.
- You can skip §8 (toggle group) and §10–11 for now.

### Not yours for now
- **Turning field values into bytes.** It needs both fields (Ariel) and the envelope (Amir). Until both are designed, payloads are just `byte[]`. We design this part together.
- Anything inside `Assets/ModularSyncedList/Core/`.

---

## Ariel: core data, in this order

Topics still to discuss before coding (topic 1, layout, and topic 2, fields, are decided and in `concept.md`):
3. One list class for both sides (server master list and client local list), and how keys reach a client's list.
4. `Add` / `Remove` / `Set` and the events they raise.
5. Lookups: declaring them, `Find`, empty sets.
6. Order: the compare function, when re-sorting happens, and whether filters and top N belong here.
7. Tests: what the first tests must prove.

No networking code in the core. The core never references NetCom or a transport.

---

## The shared contract: `ITransport` (proposal, not yet agreed)

The core sits above this interface and never sees NetCom. The networking side sits below it and never sees rows.

```csharp
public interface ITransport
{
    void Send(int clientId, byte[] data);
    void SendToAll(byte[] data);

    event Action<int, byte[]> DataReceived;   // client id, data
    event Action<int> ClientConnected;
    event Action<int> ClientDisconnected;
}
```

Questions for Amir to answer when reviewing it:
- Is one interface for both server and client right, or should the client get its own (a client only sends to the server)?
- Should the client id be a plain `int`, or a small `ClientId` type?

Once agreed, it goes into `concept.md` §7 before either of us designs on top of it. When implementation starts, it is the first code written, at `Assets/ModularSyncedList/Transport/ITransport.cs`.

---

## How we work together

- **One branch each.** Work on your own branch (e.g. `core-data`, `networking`) and merge into `main` through a pull request. Don't push straight to `main`.
- **Pull before you start,** and before editing `concept.md`.
- **`concept.md`:** edit your own sections. A change to the other person's section, or to §2 (terms), gets a heads-up first. Keep commits small so merges stay easy. Use `/update-concept` after a design discussion.
- **NetCom is never changed** (see `CLAUDE.md`).
- **Open items stay open** until both of us agree. Raise them; don't decide them silently.
- **Update this file** when the split changes.

# Work split: Ariel and Amir

Last updated: Nov 26, 2026

How the two of us work on ModularSyncedList in parallel without getting in each other's way. `concept.md` is still the single source of truth for the design; this file only says who works on what.

**Where things stand:** the design is in `concept.md`. No module code exists yet.

---

## Who does what

| | Ariel | Amir |
| --- | --- | --- |
| Area | **Core data** | **Networking** |
| What | The list class and internal row key; `Add` / `Remove` / `Set` and `FieldChanged`; lookups and `Find`; the derived `order`; fields (`Field`, `FieldList`, `IField`) | The transport and its NetCom adapter; the loopback transport for tests; the module's message envelope; NetCom's limits; the hub and routing several lists |
| Folder | `Assets/ModularSyncedList/Core/` | `Assets/ModularSyncedList/Transport/` and the NetCom adapter's folder |
| `concept.md` sections | 4, 5, 8 | 7 (networking, several lists, threading), and his items in 12 |

---

## Amir: networking, in this order

1. **Agree on `ITransport` with Ariel first** (see "The shared contract" below). It's the only code both sides touch, so nothing else starts until it's agreed.
2. **Loopback transport.** An in-process `ITransport` that connects one server and several clients inside one test, with no network. Ariel's tests will use it later to check sync.
3. **NetCom adapter.** An `ITransport` built on `NetworkServerManager` / `NetworkClientManager`. Maps NetCom's `IPEndPoint` to the module's `int` client id. Where its folder lives is **Open** in `concept.md` §12 (NetCom has no `.asmdef`).
4. **Message envelope.** One `NetMessage` type for all module traffic: list id, message type, payload. The payload is a plain `byte[]` for now. Which NetCom message id the module takes is **Open**.
5. **Measure NetCom's limits early.** How big a message can get before NetCom splits it across reads and drops it (no length framing; 2048-byte server receive buffer). Then propose: one row per message, framing inside the payload, or something else. Fields are identified by name in messages (decided, `concept.md` §4), so messages are a bit larger than with numeric ids.
6. **Hub and routing several lists.** Your proposals in `concept.md` §12. Settle them with Ariel before coding.

### What to read
- `concept.md` §1–2 (what it is, terms), §3 (principles), §7 (actions, NetCom, several lists, threading), §12 (your Open items).
- From §4 you only need: fields are identified by **name**, and the module uses **no reflection**.
- You can skip §8 (toggle group) and §10–11 for now.

### Not yours for now
- **Turning field values into bytes.** It needs both fields (Ariel) and the envelope (Amir). Until both exist, payloads are just `byte[]`. We do this part together.
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

Once agreed, it goes into `concept.md` §7 and into code at `Assets/ModularSyncedList/Transport/ITransport.cs`, in one commit, before either of us builds on it.

---

## How we work together

- **One branch each.** Work on your own branch (e.g. `core-data`, `networking`) and merge into `main` through a pull request. Don't push straight to `main`.
- **Pull before you start,** and before editing `concept.md`.
- **`concept.md`:** edit your own sections. A change to the other person's section, or to §2 (terms), gets a heads-up first. Keep commits small so merges stay easy. Use `/update-concept` after a design discussion.
- **NetCom is never changed** (see `CLAUDE.md`).
- **Open items stay open** until both of us agree. Raise them; don't decide them silently.
- **Update this file** when the split changes.

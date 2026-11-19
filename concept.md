# Modular Synced List: Concept

Last updated: Nov 19, 2026 · Status: design in progress, no code yet

This document is the single source of truth for the Modular Synced List design. Everyone working on the project, people and AI assistants alike, reads it before design or code work and updates it when a decision is made. Sections marked **Open** or **Parked** are not decided; everything else is.

---

## 1. What it is

- A reusable Unity module for server-centric lists. Every project that needs a list (for example a detection queue) builds it on this module, so new lists are quick to develop and behave consistently.
- The server holds the list, and everything passes through it. Outside systems send information only to the server, and clients connect only to the server.
- The server can change row information based on the data it receives, and clients can change it through their actions.
- Clients are not "dumb": each client holds rows as data and can filter, sort and search locally.
- Every client action is either synced through the server (it can affect other clients) or local (it changes only that client).
- "Synced" means changes go through the server. It does **not** mean every client always shows the same list; that is one possible configuration.
- The module should support as many cases as we can think of, for example the server sending to only one client, or a client searching only its own rows.

**Background:** the team leader's original PRD, Architecture and Features & Tasks documents cover more areas than this document (threading, persistence, several servers, limits, diagnostics, directory and subscriptions). They are a checklist of areas the module will eventually need. Where they differ from this document, this document is the current design.

---

## 2. Terms

| Term | Meaning |
| --- | --- |
| Module | The Modular Synced List code, the same in every project |
| List implementation | One specific list built on the module (e.g. a detection queue, a personal mission queue). It defines its row classes, compare function, conditions and actions |
| Project | The whole application. It may contain several list implementations |
| Row | One item in a list. All rows of one type share a structure, with possibly different values |
| Internal row key | A unique `int` used only by the module to identify a row, unrelated to any row data |
| Input | Something an outside system sends to the server. It may create a row, change rows, or do nothing. It is not a row and is not stored |
| Input handler | The step that matches an input against existing rows and decides: create a row, update matched rows, or ignore. Can run on the server, on a client, or both |
| Client declaration | A client action done locally at once, then reported to the server without waiting. Either **saved** or **unsaved** |
| Client request | A client action that happens only after the server sends a perform message |
| Perform message | The server's message telling clients to carry out an action |
| Master list | The server's dictionary of every row, with base values |
| Client reconnection data | Per client, on the server: which rows the client holds and its overrides |
| Override | A value one client has differently from the master list, for one field of one row |
| Unsaved view | Client UI state that resets on reconnect (search text, expanded rows) |
| Standing rules | Filters and sorts that keep applying to new rows: **fixed rules** (from the client's setup) and **runtime rules** (set by the user during the run) |
| Saved state | Values that exist only for one client and must survive a reconnect |
| Level 1 / Level 2 | Whether the server keeps nothing per client (1) or keeps client reconnection data (2) |
| Give row / Take row | Working names: the server adds an existing row to a client, or removes it from a client |

---

## 3. Design principles

1. **Support both when both are plausible.** Before asking "can this happen?", ask "could product plausibly ask for this, or could the dev team plausibly need it?" If yes, the module supports it (for example: EntityId unique or shared by several rows).
2. **The server decides who receives what.** Recipients are part of an action's definition on the server, never chosen by the requesting client.
3. **Sharing is decided by actions, not marked on parameters.** A parameter can be local in one action and shared in another. Misconfiguration is the developer's responsibility.
4. **Keep the result for things fixed once done; keep the rule for things that must keep applying to new data.** A stored filter result goes stale as soon as a new row arrives; the rule stays correct.
5. **Order is always derived.** It is the result of running the sort rule over the row data, never a stored list of positions.
6. **Last change wins, and applying the same state twice is harmless.** The server's order decides conflicts, and duplicate messages (e.g. two identical unselects) change nothing.
7. **Identity is never row data.** Rows are identified by the internal row key only.
8. **Per-client data holds differences, never copies.**

---

## 4. Rows and data

### Row data
- Each row type is a plain data class owned by the list implementation (like `DetectionRowData`).
- The module always owns only a row's identity: internal key and row type.
- Other module states (visible, selected/pinned, position) exist only when the list turns on the feature that produces them (top N, toggle group, ordering). A list without top N has no visible state.
- The module never reads row parameters directly. It calls functions the list implementation supplies, such as a compare function for sorting. The score and its scale belong to the list implementation.

### Internal row key
- Lesson from the detection queue: the detection id was both a displayed parameter and the server's dictionary key. Identity must not be row data.
- An incrementing `int` per list, always assigned by the server, even for rows created on a client. Never reused.
- Ids from outside systems (EntityId, detection id) are ordinary row data, found through a lookup.

### Shared bookkeeping
- Filtering, sorting and data management are shared code used by both the server and the client.
- All bookkeeping is written once in the module: adding, removing, moving a key in a lookup when an indexed parameter changes, re-sorting.
- Every change goes through the module (`Add`, `Remove`, `Set`), never by editing a row object directly, so lookups and order stay correct.
- Lookups map a parameter value to a `HashSet<int>` of keys, so a value can belong to one row or several.

---

## 5. The data structures

### Server: master list (one per list)
```csharp
Dictionary<int, TRow> rows;               // every row, with base values
HashSet<int> keysForNewClients;           // rows a brand-new client starts with (rows created for all)
Dictionary<TValue, HashSet<int>> lookup;  // per indexed parameter, e.g. EntityId → keys, for the server's input handler
```

### Server: client reconnection data (one per client, only at level 2)
```csharp
HashSet<int> heldKeys;                       // every row this client holds
Dictionary<int, FieldOverrides> overrides;   // per row, only the fields that differ for this client
```
- `FieldOverrides` holds only the changed fields (e.g. `IsPinned = true`). Its exact form is decided at implementation.
- No whole rows are copied, so a change to the master list is never hidden by a stale copy.
- `heldKeys` is "nothing unless included": a missing key hides a row rather than leaking one.
- When most rows go to everyone, every client's `heldKeys` is nearly the same set. Accepted for now (one `int` per row per client). A later optimization can store it as "`keysForNewClients` minus these, plus these" inside the same class, without changing the design.

### Client: local list (one per list, on each client)
```csharp
Dictionary<int, TRow> rows;               // the client's final values, already combined with its overrides
List<int> order;                          // derived by the client's sort, never sent
Dictionary<TValue, HashSet<int>> lookup;  // for local search and the client's input handler
```
- The unsaved view (search text, expanded rows) is UI state on the client only.
- `order` may hold `TRow` references instead of keys if dictionary reads inside the comparer ever become costly.

---

## 6. Inputs and the input handler

- An input is matched against existing rows on any parameter the list implementation chooses (e.g. EntityId), through the lookups.
- The list implementation supplies the conditions: per matched row, whether to update it and with which values, and separately whether a new row is created. "Always create" and "never create when matched" are just settings.
- An input may carry extra data used only for the decision and never stored on the row.
- The server processes inputs one at a time per list. This doesn't decide whether duplicates are allowed (that's the list implementation's rule); it guarantees the second input sees the row the first one created.
- **Server input handler:** its conditions decide for every client.
- **Client input handler:** its conditions apply only to that client, to what the server sends it. A list implementation can use either, or both (server first, client on top).

---

## 7. Client actions and networking

### Declarations and requests
- Each client action is configured as a declaration or a request. Configuration is per action and can differ per client (e.g. only client 1 may remove rows).
- An action's configuration controls only what a client can **send**, never what it accepts. Every client applies every change the server sends it.
- **Unsaved declaration:** changes only the unsaved view. The server keeps nothing.
- **Saved declaration:** its effect must survive a reconnect, whatever it touches (a row field, a row state, the order). It carries either saved state (the server keeps the result) or a runtime rule (the server keeps the rule).
- Each declaration is marked unsaved or saved by the list implementation when the action is defined. The same action (e.g. "sort by distance") can be either; it's the developer's choice.

### Requests
- A request means "send it, and act only if the server tells you to". The client doesn't wait, has no pending state and no timeout. A rejection is simply no perform message.
- The client doesn't change its UI for a request until the perform message arrives (don't disable a button while waiting; nothing would re-enable it).
- Before sending a perform message, the server always checks the request still makes sense; at minimum, that the row still exists. List implementations add their own checks.

### Who receives a message
- The server alone decides: the sender only, all clients, a group, one client, or everyone except the sender.
- One shared rule decides recipients, whatever caused the send: a client request, an input, or a timer.

### Messages
- The full row is sent when a row is created, when a row is given to a client, and on join or reconnect.
- Only the changed parameters are sent when a row is updated.

### Client identity
- The server identifies the sender from the network layer (the sender IP comes with every message), never from data the client writes into a message.
- The module uses its own client id internally, mapped from the connection (IP by default), in case several clients share a machine or an IP changes.

---

## 8. Toggle group

- Problem in the detection queue: the toggle changed its state on click in order to fire its event, so the sender changed before the server approved while other clients waited for the perform message.
- The module has its own toggle group: a click only raises a "toggle requested" event, and the state changes only when the group is told to apply it (locally for a declaration, on the perform message for a request). Sender and other clients go through the same path.
- Select and unselect are two separate messages. Pinning a new row sends unselect (old) then select (new), in order.
- Each client's toggle group corrects itself to its rule (e.g. only one pinned), and a correction is an ordinary synced action. A brief double pin can show for a moment and corrects itself; acceptable on a LAN.
- Rules such as one-at-a-time, allow-none or max N are group settings.

---

## 9. Several lists in one project

- A project can have several lists, e.g. a group mission queue (identical on all clients) and a personal mission queue (independent per client).
- **Taking a mission** from the group queue into your personal queue: the remove from the group queue is a synced action. Only when the server's remove actually changed the state does the server add the mission's data to the requesting client's personal queue.
- If two clients take the same mission at the same moment, only the first gets it; the second just sees the row disappear. A rare edge case on a LAN, so nothing more is shown by default.
- The module provides a server-side event when a row is removed (with the requesting client and whether the state changed), and operations to add rows for specific clients. List implementations connect them.

---

## 10. Client-specific state and reconnection

### Three kinds of client-specific things

| Kind | Examples | Server keeps it | On reconnect |
| --- | --- | --- | --- |
| Unsaved view | Search text, an expanded row | Never | Reset with the UI |
| Standing rules | "Only rows under 150 m", "sort by distance" | Fixed rules: no. Runtime rules: **Open** | Fixed rules: reapplied by the client from its setup |
| Saved state | `IsPinned` for this client only, a `ManualRank`, a personal note, rows only this client holds | Yes, as the resulting state | Restored |

### Holding vs showing
- **Holding a row** is a permanent decision: the row was sent to this client and wasn't removed for it. The server updates `heldKeys` itself whenever it sends a row, so the normal case needs no declaration. The client sends a saved declaration only for exceptions ("remove this row for me").
- **Showing a row** is a filter over held rows, applied on the client and derived every time, like the order. It is never reported to the server.
- Example: with the rule "only rows under 150 m", row 7 at 200 m stays held but hidden. If it moves to 100 m (even while the client is disconnected), it shows, including right after a reconnect.

### Order
- Order is derived from row data and the sort rule. A saved declaration that affects order changes row data (`IsPinned`, `ManualRank`), never a position. "Pinned rows first" is part of the sort rule, usually a fixed rule.
- For per-client lists, **the client sorts**. The server keeps no order for a client, and sorts only when it needs the order for a decision of its own, at that moment, keeping nothing.
- **Top N** can run on the server only when everything that decides it is identical on every client: the same rows, the same values on every parameter the sort and filter use, and the same rules. If anything can differ per client (a removed row, a local pin, an override, a client filter), top N runs on the client.

### Levels (derived automatically, no manual setting)
- **Level 1:** the list has no saved declarations and the server never sends list changes to fewer than all clients. The server keeps only the master list.
- **Level 2:** the list has saved declarations, or the server can send list changes to fewer than all clients. The module automatically creates client reconnection data for each client and wires the networking.

### Where overrides come from
1. **Saved declarations from a client**, for that client only.
2. **Value changes the server sends to only some holders of a row** (most often a client request whose perform message goes to some clients). The server writes these overrides itself; recipients send nothing back.

### Server flow

| Event | Master list | `keysForNewClients` | Client's `heldKeys` | Client's overrides |
| --- | --- | --- | --- | --- |
| Row created for all | Add row | Add key | Add key for every known client, connected or not | — |
| Row created for some | Add row | — | Add key for those clients | — |
| Give row (existing row to a client) | — | — | Add key, send full row | — |
| Value change sent to all holders | Update row | — | — | Clear that field on that row for everyone (last change wins) |
| Value change sent to some holders | — | — | — | Set the field for each recipient |
| Saved declaration changing a value | — | — | — | Set the field for that client |
| Saved declaration "remove for me" | — | — | Remove key | Clear that row's overrides |
| Take row (server removes a row from a client) | — | — | Remove key, tell the client | Clear that row's overrides |
| Row removed for all | Remove row | Remove key | Remove key everywhere | Clear that row everywhere |
| New client joins | — | — | Created from `keysForNewClients` | Empty |
| Client reconnects | — | — | Used to build its full send | Applied to each row |

### A client reacting differently to a perform message
- A client's own conditions can decide not to apply a server change the same way as others. Example: the server sends "row 5 `IsPinned = false`" to all and clears overrides; client B keeps its pin locally and sends a saved declaration; the server sets B's override back. B's declaration is caused by the server's change, so it always arrives after it, and the final state matches what B shows.

### Reconnect flow
1. The server recognizes the client (how: decided at implementation).
2. For each key in the client's `heldKeys`, the server takes the row from the master list, applies the client's overrides, and sends the result as a full send. At level 1, it sends the master list.
3. The client applies its standing rules and sort, and derives its order and what it shows.
4. The unsaved view starts reset.

### Lifetime of client reconnection data (direction agreed, details later)
- After a client disconnects, the server keeps updating its data and starts a timer. If the client doesn't return within that time, its data is deleted.
- A client returning after its data was deleted is treated as brand new (starts from `keysForNewClients`, no overrides).

---

## 11. Example: the detection queue from the last project

One project's implementation, used as a test case for the module, **not** its blueprint.

**Product view**
- The detection list holds detection rows with the same structure and different values. Rows are created and updated from detections sent by several outside systems. Not every detection creates a row: if a row with the same EntityId exists, a condition decides whether it is updated.
- Displayed parameters: detection id (unique, +1 per new row), distance from the vehicle, and more values from the detection.
- Row buttons: one makes the vehicle act, one removes the row, and more. Clicking a row pins it: it changes color and moves to the top. Only one row can be pinned; clicking it again unpins it, and clicking another row moves the pin.
- At most 10 rows are shown, ordered by a priority score calculated from each row's parameters. Hidden rows can appear later when another row disappears or a score changes.
- Every client shows the exact same list at all times, including the pinned row.

**How it was built**
- The server connected to the outside systems, held all sorting logic (custom comparer, descending priority), and changed row values on client requests or outside data.
- The client was deliberately "dumb": no list or order, only a `DetectionIdToRowManager` mapping detection id to its row game object.
- A button press sent a request; the server sent an execute to all clients, plus an extra "perform this action" message only to the requesting client (e.g. the vehicle action), so it ran once.
- On connect, the server sent the full sorted list. It told clients about every created row, even hidden ones; the client kept hidden rows disabled, and the server sent which rows to turn on or off by comparing the last visible list with the new one.
- Data: a `List<DetectionRowData>` sorted with a custom comparer, plus dictionaries keyed by detection id and EntityId. Value changes were easy (rows were reference types); adding, removing and changing an EntityId meant updating every structure by hand.

**What the module takes from it**
- Keep: one row data class, a sorted list plus dictionaries, a custom comparer, top N, full state on connect, targeted messages.
- Change: an internal key instead of the detection id, lookups that allow several rows per value, bookkeeping written once in the module, a client that holds data and can act locally, and a toggle group that doesn't change state on click.

---

## 12. Open, parked and out of scope

### Next topic
- **One list holding two different row types.** Options: a common base type with casts; one set of structures per type with a shared key counter; a module-owned record per row plus typed data stored per type; a superset row class; two lists combined on the client. Not decided.

### Open
- **Runtime rules storage:** where the server keeps runtime rules that must survive a reconnect. Proposal (not approved): store only their values per client (e.g. "max distance: on, 150"), as part of client reconnection data, separate from row data.
- **Overriding a parameter the server matches on** (e.g. a client overrides EntityId, but the server's input handler matches on the master value). Two options: (a) the module forbids overrides on matched parameters by default; (b) overrides are allowed, and a client that can override a matched parameter doesn't receive the server's matched results; the server forwards it the input instead, and its own client input handler matches against its own values.
- **A master row no client holds anymore:** keep it (so the input handler can still match it) or delete it.
- **A client input handler skipping a row:** on reconnect the skip isn't replayed. Probably expressed either as "remove for me" (permanent) or as a standing rule (re-evaluated); the list implementation chooses.
- **Where per-client action configuration comes from,** and whether the server checks a client's authority before processing a request.
- **Whether the server keeps an order for a shared list it sorts itself** (as the detection queue did for top 10), or sorts on demand.
- **Networking:** which library carries the messages, and whether the server and clients build from the same Unity project.
- **UI binding:** whether the module owns the binding from rows to UI game objects, or only data and events.
- **Name:** "Modular Synced List" is the current name; "Hub List" was the main alternative considered.

### Left for implementation
- How a client gets a key for a row it creates (wait for the server, or a temporary key).
- How the server recognizes a reconnecting client.
- The exact timer and deletion rule for client reconnection data.
- The exact form of `FieldOverrides`.

### Parked
- **Saved declaration racing a server change during a crash:** the server clears a client's override as part of a change to all, and the client crashes before it can react and restore it, so after reconnect it shows the server's value. Rare; revisit after the project is done. Fix ideas: don't clear overrides on fields where the client may react, or have the client re-run its reactions over the full send on reconnect.
- Optimistic actions (act at once, roll back on rejection).
- An explicit rejection message for requests.
- Recording that an input didn't create a row because a matching row already existed.
- Server-side filtering per client (the server sends only what passes a client's conditions).
- Storing overrides once per group instead of per client.
- The `heldKeys` "default minus/plus" optimization.

### Out of scope for now
- Server restart: all data, including client reconnection data, is lost.

---

## 13. Working agreements

These apply to everyone working on the design, including AI assistants.

- **Use this document's vocabulary.** Don't introduce new terms in code or discussion without agreeing on them and adding them to section 2.
- **Keep the module and any one list implementation separate.** An example (like the detection queue) is a test case, not the shape of the module.
- **Apply principle 1 before asking questions:** if both answers are plausible for some product, the module supports both, and there's nothing to ask.
- **When someone restates a design point to check understanding,** answer first with "same", "corrected" or "partly", then explain.
- **Explain complex behavior fully,** with a concrete example and code-level terms (`Dictionary`, `HashSet`, `Sort`), and split long explanations into parts.
- **Give an opinion with every question** where there's a reasonable default.

## 14. Updating this document

- Update it in the same commit or pull request as the decision or code it describes.
- Move items from **Open** to the relevant section when they're decided, and record anything newly undecided under **Open** rather than leaving it implied.
- Bump "Last updated" at the top.

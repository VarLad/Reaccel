# The Reaccel Protocol

Version 0.1.0 — Draft

The Reaccel protocol lets an editor drive a reactive notebook. A notebook is a file of
code split into **blocks**. The server runs the code, works out which blocks depend on
which, and re-runs the affected blocks when one changes.

The protocol is designed to solve the same narrow-waist problem as
[nREPL](../spec.md): there are N languages and M editors, and writing N×M adapters is
wasteful. Like nREPL, Reaccel achieves its integration by *running* project code rather
than analysing it statically. Unlike nREPL, the unit is not a request but a **block** —
code that persists, has a name, and can be edited and re-run.

Reaccel is language-agnostic. It has been implemented twice, for Clojure and for Lua, and
both are driven by the same client. The wire is portable; what a block *means* is not
(see [Language-specific behaviour](#language-specific-behaviour)).

**Draft.** This document describes version 0.1.0. It is written to be implemented, not
just read; every message shape shown here was taken from a working server.

## Protocol

The protocol exchanges [JSON](https://www.json.org) messages over a WebSocket. nREPL's
default is bencoded messages over a socket; Reaccel uses JSON over WebSocket because a
notebook client is a browser program, which has WebSocket built in and no bencode
library. Other encodings are not specified.

Every message sent by the client is a **request** and must have an `op` field, for
operation. Every message sent by the server is either a **reply** or a
**notification**.

A request may produce any number of reply messages. The request is considered active
until a reply arrives whose `status` field is a list containing the string `done`:

```
// client -> server
{"op": "eval", "id": "r1", "blocks": ["a"]}

// client <- server
{"id": "r1", "ran": ["a", "b"]}
{"id": "r1", "status": ["done"]}
```

Requests should have an `id` field, and replies to that request must carry the same
`id`. The only requirement on `id` is that it is a string unique to the life of the
server process; in this document they are short identifiers for readability. A client
may use any string, including a UUID.

Notifications are messages the server sends without being asked. They carry a `type`
field instead of an `id`, and expect no response:

```
// client <- server
{"type": "state", "tick": 12, "notebook": {...}}
```

When the server encounters an error in itself — as opposed to in the code it was asked
to run — it should reply with `err` describing the problem and a `status` containing
`server-error` and `done`.

## Blocks

A block is a named, versioned unit of code with server-computed dependencies.

```json
{"block": "3f9a2c11",
 "code": "(def x 41)",
 "rev": 3,
 "writes": ["x"],
 "reads": [],
 "deps": [],
 "child": ["7c2e9a11"],
 "value": "x => 41",
 "defs": {"x": "41"},
 "queued": false,
 "running": false,
 "errored": false,
 "disabled": false,
 "depends_on_disabled": false,
 "interrupted": false,
 "error": null,
 "error-kind": null,
 "error-refs": null}
```

`code` is the block's source. It is the only field the client controls.

`writes` and `reads` are **derived by the server**, never declared by the client. If a
client could set them, the graph would be unverifiable. `writes` lists the bindings a
block creates in the shared environment; `reads` lists the names it reads that it does
not itself bind first. Both are language-specific strings, and their meaning is fixed by
the language (see [Language-specific behaviour](#language-specific-behaviour)).

`deps` are the blocks that must run before this one, derived from `writes` and `reads`.
`child` is the inverse: the blocks that re-run when this one changes. Both are
server-maintained.

`rev` increments on every edit. A result is the product of a particular `rev`; if the
code changes while a block is running, that result is discarded (see
[Consistency](#consistency)).

`value` is a rendering of what the block last produced, as a string, for display. It is
**not** a machine-readable representation. `defs` maps each name in `writes` to the
value it was given, which is usually what a user wants to see: for `(def x 41)` the
`value` is the rendering of the `def` form, but `defs` is `{"x": "41"}`.

The boolean fields — `queued`, `running`, `errored`, `disabled`, `depends_on_disabled`,
`interrupted` — describe the block's state. They are orthogonal, not an enumeration: a
block can be both `errored` and `depends_on_disabled`. A client must derive what it
displays from the combination, not store an enum. They are described in
[Block state](#block-state).

The remaining fields are `error` (a message for the user), `error-kind` (`analysis`,
`parent`, or `runtime`), and `error-refs` (block ids the error message refers to, so a
client can render links instead of printing identifiers).

## Operations

The first four operations — `describe`, `create`, `edit`, and `eval` — are the absolute
minimum a server needs to support. The `notebook/*` operations are required for a server
that stores notebooks in files, which is strongly recommended.

### `describe` op

The client should send a `describe` request upon connecting so it knows which
operations the server supports. Servers should not require clients to make this request
first.

```
// client -> server
{"op": "describe", "id": "d1", "client": "Reaccel example client 1.0.0"}
```

The `client` field is for debugging; the server should not behave differently based on
it.

The response must have a list of `ops` supported by the server, and should have a
`versions` dictionary giving the implementation's version and the language version:

```
// client <- server
{"id": "d1",
 "ops": ["describe", "create", "edit", "delete", "eval", "run", "state",
         "dependents", "disable", "enable", "interrupt", "close",
         "notebook/open", "notebook/create", "notebook/save", "notebook/info"],
 "versions": {"reaccel": "0.1.0", "clojure": "1.12.0"},
 "status": ["done"]}
```

The language name in `versions` is how a client learns which language it is editing —
`clojure`, `lua`, and so on. There is no separate field for this. A client that does not
recognise the language still works; it simply cannot offer syntax-specific editing.

### `create` op

Registers a block. This does not run it.

```
// client -> server
{"op": "create", "id": "c1", "block": "3f9a2c11", "code": "(def x 41)", "position": 0}

// client <- server
{"id": "c1", "block": "3f9a2c11"}
{"id": "c1", "status": ["done"]}
```

`block` is optional; if omitted, the server chooses an id and returns it. `position` is
optional and gives the block's index in notebook order; if omitted, the block is
appended. Order affects only display and the initial run order, never dependencies.

`create` is idempotent with respect to a block that already exists: re-creating an
existing id replaces its code rather than adding a second copy.

### `edit` op

Replaces a block's code, increments its `rev`, and re-analyzes it.

```
// client -> server
{"op": "edit", "id": "e1", "block": "3f9a2c11", "code": "(def x 100)"}

// client <- server
{"id": "e1", "block": "3f9a2c11", "rev": 4}
{"id": "e1", "status": ["done"]}
```

`edit` does not run anything. This separation from `eval` is the point of the protocol:
the server can analyze code *before* it runs, which is what makes the dependency graph
available in advance of execution.

If the block does not exist, the reply's status is `unknown-block`.

### `eval` op

Runs the named blocks and, after each, the blocks that depend on it.

```
// client -> server
{"op": "eval", "id": "v1", "blocks": ["3f9a2c11"]}

// client <- server
{"id": "v1", "ran": ["3f9a2c11", "7c2e9a11"]}
{"id": "v1", "status": ["done"]}
```

The blocks are run in dependency order, which the server computes; the order is not
necessarily the order given. `ran` lists what actually executed, in the order it
executed.

Code that prints produces `out` messages:

```
{"id": "v1", "block": "3f9a2c11", "out": "hello"}
```

As in nREPL, output may arrive with an `id` for a request already considered done, and
clients should be prepared for output missing an `id` entirely.

If a block raises, it gets `errored` set and an `error`. The run continues: one failing
block does not abort its siblings. Blocks that depend on a failed block do not run, and
are marked as having a parent error (see [Errors](#errors)).

`eval` may be sent with a block id that does not exist; the run reports no blocks.

### `run` op

A convenience form: `{"op": "run", "block": "3f9a2c11"}` is equivalent to
`eval` with that single block, and also runs its transitive dependencies first.

### `state` op

Returns the whole notebook. Clients use this to synchronize, and may poll it.

```
// client -> server
{"op": "state", "id": "s1"}

// client <- server
{"id": "s1",
 "notebook": {"blocks": [...], "order": ["3f9a2c11", "7c2e9a11"]}}
{"id": "s1", "status": ["done"]}
```

The server also pushes this shape unprompted as a `state` notification; see
[Notifications](#notifications).

### `delete` op

Removes a block. Its names are unbound, which may make dependent blocks fail.

```
// client -> server
{"op": "delete", "id": "x1", "block": "3f9a2c11"}
```

### `dependents` op

Returns the blocks that depend on a given block. With `"transitive": true`, returns the
whole downstream closure.

```
// client -> server
{"op": "dependents", "id": "p1", "block": "3f9a2c11", "transitive": true}

// client <- server
{"id": "p1", "block": "3f9a2c11", "dependents": ["7c2e9a11"]}
```

### `disable` and `enable` ops

A disabled block does not run, and neither do the blocks that depend on it, which
acquire `depends_on_disabled`. Disabling is for temporarily removing a block from the
reactive graph without deleting its code.

```
// client -> server
{"op": "disable", "id": "d1", "block": "3f9a2c11"}
```

`enable` reverses it and re-analyzes the block, since its inputs may have changed while
it was off.

### `interrupt` op

Asks the server to stop running code. The `interrupt-id` field names the request to
interrupt; if omitted, the most recent request in the session.

```
// client <- server
{"id": "3f9a2c11", "status": ["interrupted", "done"]}
```

Interruption is best-effort. A server that cannot interrupt should reply with
`interrupted` and `done` immediately and let the code finish.

### `close` op

Ends the session. The server may close the socket after replying. It should remain
running to accept other clients.

### Unknown operations

If a server receives a request with an `op` it does not recognise, it must reply with a
`status` containing `unknown-op` and `done`.

```
{"id": "q1", "status": ["unknown-op", "done"]}
```

## Notebooks

A notebook is a **file** whose extension is the language it holds: `.clj`, `.lua`, `.jl`,
`.py`. There is no container format. Opening a notebook in an editor with no Reaccel
support shows ordinary, runnable source, and a notebook can be committed, diffed, and
reviewed like any other file.

### File layout

Blocks are delimited by comment lines, so the file stays valid source:

```clojure
;;; Reaccel notebook
;;; include: lib/base.clj

;;; ┌─ cell
(def x 41)

;;; ┌─ cell
(def y (inc x))
```

A block runs from its marker to the next marker, or to the end of the file. Trailing
blank lines belong to the file, not to the block, and are stripped when reading.

The marker is the language's line comment, whitespace, and the rune `┌─`. Everything
after the rune on that line is a comment: `┌─ cell` and `┌─ block` are the same marker.
Use `cell` in new files; the rune is what matters, and a parser must accept any trailing
text so that prettier or a formatter cannot change the file's meaning.

| language | comment |
| --- | --- |
| Clojure | `;;;` |
| Lua | `---` |
| Julia, Python, Ruby, shell | `#` |

A closing marker `└─` is optional and cosmetic. It exists so a human can scan the file;
the parser ignores it. A file with no closers is valid.

The header is optional. `Reaccel notebook` identifies the file; `include:` lines compose
notebooks (see below). Both are comments and have no effect on execution.

The file stores no block ids. Ids are assigned when a notebook is opened, so reopening a
file gives it new ids. A client must therefore key its own bookkeeping on position or
content, not on the id it saw in a previous session.

File order is block order. There is no separate index.

### Includes

An `include:` line names another notebook to run **before** this one's blocks, in the
order listed:

```clojure
;;; Reaccel notebook
;;; include: lib/base.clj
;;; include: lib/helpers.clj

;;; ┌─ block
(def report (render base-value))
```

`lib/base.clj` runs first, then `lib/helpers.clj`, then this notebook's blocks. Paths are
relative to the including notebook's own directory.

Four rules:

1. **Includes do not nest.** If an included notebook has `include:` lines of its own,
   they are ignored. This keeps the run order a flat list that a reader can predict from
   one file, and makes a cycle between notebooks impossible to express.
2. **A missing include is an error.** The run fails with `include-not-found`; the
   notebook does not partially run.
3. **Names are shared.** Included blocks and local blocks run in the same environment, so
   the single-writer rule applies across them. If `lib/base.clj` defines `x` and this
   notebook also defines `x`, the run is rejected. This is the most common surprise when
   composing notebooks, and it is intentional: the alternative is a silent overwrite.
4. **Included blocks are not editable through the includer.** They belong to the included
   notebook. `state` shows only the open notebook's blocks.

### `notebook/list` op

Lists notebooks on disk under a directory.

```
// client -> server
{"op": "notebook/list", "id": "l1", "dir": "."}

// client <- server
{"id": "l1",
 "notebooks": [{"path": "demo/main.clj", "name": "main.clj",
                "language": "clojure", "blocks": 2, "includes": 1}]}
```

Paths are relative to the server's working directory. The server should skip build and
vendored directories.

### `notebook/open` op

Loads a notebook file as the session's notebook, replacing whatever was open.

```
// client -> server
{"op": "notebook/open", "id": "o1", "path": "demo/main.clj"}

// client <- server
{"id": "o1",
 "notebook": {"path": "demo/main.clj", "language": "clojure", "dirty": false,
              "includes": ["lib/base.clj"], "blocks": [...]},
 "ids": ["3f9a2c11", "7c2e9a11"]}
```

`ids` gives the block ids assigned on open, in file order, so a client can adopt them
without diffing. Opening does not run anything; blocks are analyzed but not queued.

If the path does not exist, the status is `notebook-not-found`.

### `notebook/create` op

Creates a notebook file and opens it.

```
// client -> server
{"op": "notebook/create", "id": "n1", "path": "demo/scratch.clj", "language": "clojure"}

// client <- server
{"id": "n1", "path": "/abs/path/demo/scratch.clj", "notebook": {...}}
```

If `path` has no recognised extension, the server appends one for `language`. If the file
already exists, the status is `exists` and nothing is overwritten.

A new notebook contains one empty block.

### `notebook/save` op

Writes the open notebook to disk.

```
// client -> server
{"op": "notebook/save", "id": "s1"}

// client -> server, saving under a new name
{"op": "notebook/save", "id": "s2", "path": "demo/renamed.clj"}

// client <- server
{"id": "s1", "path": "/abs/path/demo/main.clj", "notebook": {...}}
```

Saving never runs code. A path different from the open notebook's is a save-as: the file
is written there and becomes the session's path.

If the session has no path and none is given, the status is `no-path`. A client should
prompt for one.

### `notebook/include` and `notebook/uninclude` ops

Add or remove an `include:` line.

```
// client -> server
{"op": "notebook/include", "id": "i1", "path": "lib/util.clj"}

// client <- server
{"id": "i1", "notebook": {"path": "...", "includes": ["lib/util.clj"], "dirty": true}}
```

Includes are appended in the order added. There is no reordering operation; to move one,
remove it and add it again. Adding an include that does not resolve is `include-not-found`
and the include is not added.

### `notebook/info` op

Returns the session's notebook metadata without the blocks.

```
// client <- server
{"id": "i1",
 "notebook": {"path": "demo/main.clj", "language": "clojure",
              "dirty": true, "includes": ["lib/base.clj"]},
 "status": ["done"]}
```

`dirty` is true when there are edits that have not been saved.

## Dependencies

Dependencies are computed from `writes` and `reads`, and only from those. The server does
not record what happened at runtime; a block's graph position must be a function of its
code alone.

This matters for a specific reason. If the graph changed depending on what actually ran,
then the graph a notebook *should* have had would differ from the graph it *did* have, and
the notebook's state would depend on execution history. A notebook must be reproducible:
opening the same file must produce the same graph.

The consequence is that a server must analyse code it cannot parse conservatively — over-
reporting a read causes a redundant re-run, while under-reporting causes a silently stale
result. Servers should prefer a redundant re-run.

### Single writer

**A name may be written by at most one block, within one evaluation.**

- `A: (def x 1)` and `B: (def x 2)` evaluated together → `multi-writer`, the run is
  rejected.
- `A: (def x 1)` and `B: (def x 2)` both present, but only `A` (or an unrelated `C`) is
  run → fine. The clash is real but irrelevant to that evaluation.
- `A: (def x 1)` followed by `(def x (+ x 1))` in the *same* block → fine, and `x` is
  not a free read, because the block binds it before reading it.
- `A: (def x 1)` and `B: (def x (+ x 1))` → `multi-writer`: two blocks own `x`.

The rule exists because it makes the question "which value does a reader see?" have one
answer. Without it, `(def y (inc x))` has no defined input.

The scope is the evaluation, not the notebook. Two blocks that both define `y` do not
freeze unrelated blocks. A notebook-wide clash is reported when the code is edited, so
the author sees it, but it only blocks runs that involve both.

### Cycles

A dependency cycle is an error. The run is refused with `cyclic-reference`, and every
block in the cycle is marked `errored` with that error.

A cycle is not iterated to a fixed point. Iteration would make the final values depend on
how many rounds the server happened to run, which is exactly the reproducibility the
graph exists to provide.

## Evaluation

Blocks run in a single shared environment, so one block's definitions are visible to the
next. That shared environment is the notebook's state, and it is why `single-writer`
exists.

### One evaluation at a time

A server runs at most one block at a time, server-wide. A block cannot start while
another is running.

This is not a simplification for implementers; it is what makes evaluation
comprehensible. Concurrent blocks that both touch shared state have no defined outcome,
and a notebook that behaves differently depending on scheduling is not reproducible.

Clients may send several evaluation requests; the server queues them. `queued` is true
while a block is waiting.

### Order

Within one evaluation, blocks run in dependency order. `ran` reports what actually
executed, in order.

Blocks that share no dependency may be run in any order; the server chooses. A client
must not assume the order of `ran` beyond the dependency constraints.

## Block state

A block's state is a set of independent booleans, not an enumeration. A client derives
what to display:

| display | condition |
| --- | --- |
| running | `running` |
| queued | `queued` and not `running` |
| disabled | `disabled` or `depends_on_disabled` |
| errored | `errored` |
| idle | none of the above |

`errored` and `depends_on_disabled` can both be true, which is why this is not an enum.

`disabled` is set by the client and means "do not run this block". `depends_on_disabled`
is set by the server and means "an upstream block is disabled". Both stop execution;
only the first is a user decision.

`interrupted` means the last run was stopped rather than failing. It is distinct from
`errored` so a client can tell "you stopped me" from "your code is broken". A dependent
of an interrupted block is not failed, because nothing is wrong with it.

## Errors

Errors are reported in two places: in the reply's `status`, and on the block itself.

A block's `error` is a message for a person; `error-kind` says what sort of problem it is.
The kinds are ordered by precedence — the first that applies wins:

1. **`cyclic-reference`** — the block is part of a dependency cycle.
2. **`multi-writer`** — another block defines a name this one defines.
3. **`analysis`** — the code cannot be parsed or analyzed.
4. **`parent`** — a block this one depends on is errored.
5. **`runtime`** — the code ran and raised.

A parent error takes precedence over the dependent's own runtime error because the
dependent's failure is a symptom. A block reading an undefined name fails with an error
about that name; the useful message is "an error occurred in a parent block".

The message for a parent error is fixed, so clients can rely on it:

```
An error has occurred in a parent block
```

Errors that name other blocks must not embed their ids in the text. The server sends
readable prose plus `error-refs`, a list of block ids, so a client can render links:

```
{"error": "redefinition not allowed: x is also defined here",
 "error-refs": ["7c2e9a11"]}
```

A client that cannot render links still shows a readable message.

### Status tokens

A reply's `status` is a list. It must contain `done` to terminate the request, and may
contain one of:

| token | meaning |
| --- | --- |
| `done` | the request completed |
| `unknown-op` | the server does not support this operation |
| `unknown-block` | no such block |
| `server-error` | the server itself failed |
| `multi-writer` | a name is defined twice in this run |
| `cyclic-reference` | the blocks to run form a cycle |
| `interrupted` | the run was interrupted |
| `notebook-not-found` | no such notebook file |
| `exists` | `notebook/create` would overwrite |
| `no-path` | `notebook/save` has nowhere to save |
| `include-not-found` | an include does not resolve |
| `bad-path` | a path outside the server's working directory |

## Notifications

The server pushes two kinds of message. Neither expects a reply.

### `state`

Sent whenever the notebook changes: after `create`, `edit`, `delete`, a notebook
operation, and after an evaluation completes.

```json
{"type": "state", "tick": 12,
 "notebook": {"blocks": [...], "order": ["3f9a2c11", "7c2e9a11"]}}
```

`tick` increases monotonically on every change. A client uses it to discard out-of-order
or duplicate pushes.

A server sends the whole notebook rather than a diff. A notebook is small, and a diffing
protocol costs more in specification than it saves in bytes. Implementations may send
diffs internally, but the message shape is a full notebook.

### `progress`

Sent while a block is about to run and again when it finishes. Clients use it to animate
a running block before the final `state` push.

```json
{"type": "progress", "block": "3f9a2c11", "queued": true, "running": false}
```

## Sessions

A session has one open notebook and one set of blocks. Servers may support several
sessions; `clone` and `close` work as in nREPL.

A client should send `describe` and `state` on connecting. The server also pushes `state`
on connection, but a client that attaches its message handler late may miss it, so
pulling is the reliable path.

## Design notes

A few decisions are worth explaining, because they look arbitrary otherwise.

**Why not let the server build the graph it wants?** Because then the client cannot check
it. A block's `code` is sent by the client and echoed back; `writes` and `reads` are
derived and can be recomputed by inspection. `error-refs` lets a client verify which
blocks an error refers to. Nothing in the protocol asks a client to trust a claim it
cannot confirm.

**Why is `edit` separate from `eval`?** nREPL conflates "here is code" with "run it",
because code only exists inside an eval. Reaccel keeps code in the server, so the server can
analyze it before running it. That is the capability nREPL cannot have, and it is what
makes the graph available in advance.

**Why one evaluation at a time?** See [Evaluation](#evaluation). Concurrency over shared
state is not reproducible.

**Why blocks rather than files?** Because a file is all-or-nothing. Editing one function
should re-run the blocks that use it, not the whole file. The file is where the notebook
is *stored*; blocks are what the protocol operates on.

## Language-specific behaviour

The protocol fixes the message shapes; it does not fix what a block means. Each server
decides, and clients that care must discover it from `describe`'s `versions`.

### Clojure

- A block writes a var when it uses `def`, `defn`, `defmacro`, `defonce`, `defmulti`,
  `defprotocol`, `defrecord`, `deftype`, `declare`, and similar forms.
- A block reads a name freely when it references a symbol it has not bound itself.
- Names resolving in `clojure.core` are not dependencies: no block owns `inc`.
- `let`, `fn`, `loop`, `for`, `doseq`, `binding` and destructuring introduce locals,
  which never appear in `reads`.

### Lua

- A block writes a **global** when it assigns a name not introduced by `local`.
- `local x = 1` is not a write; `x = 1` is.
- `function f() end` writes `f`. `local function f() end` does not.
- Function parameters and loop variables are locals.

### What cannot be standardised

Whether a block can be analyzed at all depends on the language. Code that builds code at
runtime — Clojure's `eval`, Julia's macros, Python's `exec` — cannot be analyzed
statically. A server may report such a block as unanalyzable rather than guess, in which
case the block runs but does not participate in the graph.

The protocol permits this. What it does not permit is a server silently under-reporting
dependencies, because that produces a notebook that looks correct and is stale.

## Conformance

A server must support `describe`, `create`, `edit`, and `eval`. A server that stores
notebooks in files must also support the `notebook/*` operations.

Two optional capabilities exist, and a server should advertise them in `describe`:

- **`dependents`** — the forward closure can be queried rather than derived by the client.
- **`run`** — a block can be run together with its dependencies in one operation.

A server that supports none of the optional operations is still a Reaccel server. A client
must work with the required set alone.

## Example session

```
// client -> server
{"op": "describe", "id": "1", "client": "example 1.0.0"}

// client <- server
{"id": "1", "ops": ["describe", "create", "edit", "eval", "state"],
 "versions": {"reaccel": "0.1.0", "clojure": "1.12.0"}, "status": ["done"]}

// client -> server
{"op": "create", "id": "2", "block": "a", "code": "(def x 41)", "position": 0}

// client <- server
{"id": "2", "block": "a"}
{"id": "2", "status": ["done"]}

// client -> server
{"op": "create", "id": "3", "block": "b", "code": "(def y (inc x))", "position": 1}

// client <- server
{"type": "state", "tick": 3,
 "notebook": {"blocks": [
   {"block": "a", "code": "(def x 41)", "writes": ["x"], "reads": [],
    "deps": [], "child": ["b"], "rev": 0, ...},
   {"block": "b", "code": "(def y (inc x))", "writes": ["y"], "reads": ["x"],
    "deps": ["a"], "child": [], "rev": 0, ...}],
  "order": ["a", "b"]}}

// client -> server
{"op": "eval", "id": "4", "blocks": ["a"]}

// client <- server
{"id": "4", "ran": ["a", "b"]}
{"id": "4", "status": ["done"]}

// client <- server
{"type": "state", "tick": 4,
 "notebook": {"blocks": [
   {"block": "a", "value": "x => 41", "defs": {"x": "41"}, ...},
   {"block": "b", "value": "y => 42", "defs": {"y": "42"}, ...}], ...}}

// client -> server
{"op": "edit", "id": "5", "block": "a", "code": "(def x 100)"}

// client -> server
{"op": "eval", "id": "6", "blocks": ["a"]}

// client <- server
{"id": "6", "ran": ["a", "b"]}
{"id": "6", "status": ["done"]}

// client <- server
{"type": "state", "tick": 6,
 "notebook": {"blocks": [
   {"block": "a", "value": "x => 100", ...},
   {"block": "b", "value": "y => 101", ...}], ...}}
```

Editing `a` and evaluating it re-ran `b`, because `b` reads `x`.

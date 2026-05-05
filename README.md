# notes

Backend engineering notes. A working set of notes for thinking through
ideas and coming back to them later. Revised in place rather than logged
by date.

## Layout

- `data/` — single-store internals: transactions, isolation, MVCC, JPA,
  Redis data structures, cache patterns.
- `system-design/` — cross-component coordination: idempotency, sagas,
  outbox, distributed locks, messaging, case analyses.
- `algorithm/` — algorithm and data-structure notes.
- Top-level `.md` files — standalone notes that don't yet justify a folder.

Folders exist when a topic has earned one. New notes start at the root or
in an existing folder; a new folder is created only when several notes
cluster around the same theme.

## Conventions

- Folders and files use `kebab-case`. One topic per file.
- Notes describe the current understanding, not a session log. No dated
  sections; revise existing prose instead of appending.
- Each note should make the trade-offs explicit and answer *when to use
  what*. The shape of the note follows the topic — patterns, internals,
  and case analyses don't read the same way.
- Cross-references use relative links.
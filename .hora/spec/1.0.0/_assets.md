# What stage 0 read — 1.0.0

Read at breadth on 2026-08-30, against a working tree with no commits yet.

## Repositories

| Repository | Exists | Read |
|---|---|---|
| `expense-note-backend` | no | — |
| `expense-note-frontend-*` | no | — |

**Nothing is implemented.** The working directory holds the hora repository alone: the kit,
its documents, and an empty `specs/1.0.0/`. There is no code, no migration, no schema and no
screen to read, so nothing in this version is drafted as a check against a running system.

## Drop-off directories

| Directory | Contents |
|---|---|
| `specs/1.0.0/request/` | empty — the request arrived in conversation instead |
| `specs/1.0.0/sources/` | empty |
| `specs/1.0.0/annex/` | empty |

`Sources` and `Annex` are therefore both empty in `spec.md`.

## Read but not part of the specification

| File | Why it is not a source |
|---|---|
| `docs/**`, `about-boilerplate.md` | the boilerplate's own documents. They describe the kit, not this product |
| `docs/stack/**` | the stack handbook — read as the catalog of origins, middleware and API kinds that stages 2 to 5 write against. It constrains the design; it states no requirement of this product |

## The request

Stated in conversation, not in a file: **an expense note application — sign in, record an
expense, see the month's entries with a total, correct or remove your own.** Stage 1 works
through it; nothing in it is spec text until a stage drafts it and it is approved.

## Divergence

**None.** `_divergence.md` is not written: documents and code have to both exist for the two to
disagree, and neither does.

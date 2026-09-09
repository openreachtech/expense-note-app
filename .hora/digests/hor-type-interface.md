# hor-type-interface
<!-- hora-skills-ort-renchan 0.1.0 -->
<!-- source: .claude/skills/hor-type-interface/ -->

**Read the source above whenever this leaves a question open.**

## Grand principle: one file per entity, merged into one global namespace

Every model and every resolver gets its **own `.d.ts` file**, each declaring types into a **shared global namespace**. TypeScript's **declaration merging** combines the same-named `namespace` blocks from every file into one.

- A monolithic `model.d.ts` / `graphql.d.ts` is forbidden — one file per entity. **Adding a model = adding a file.**
- `declare global { namespace model { … } }` in `User.d.ts` and the same block in `Order.d.ts` **merge** into a single `model` namespace. Consumers write `model.User` / `model.Order` regardless of which file declared them.
- **Wiring**: the project's `tsconfig` / `jsconfig` `include` must cover the `types/**` directory so these ambient declarations are picked up project-wide.

**Sample code follows the project's lint style**: `.d.ts` uses no semicolons, 2-space indent, and interface members are one-per-line (no trailing comma or semicolon). Comments are English for structural notes; domain notes match the surrounding language.

## Naming & placement (quick reference)

| Kind | File | Namespace | Interface names | Access |
| --- | --- | --- | --- | --- |
| Model | `types/models/<ModelName>.d.ts` | `model` | `<ModelName>` | `model.<ModelName>` |
| Resolver | `types/resolvers/<category>/<resolverName>.d.ts` | `graphql.<category>` | `<Resolver>Input`, `<Resolver>Result` | `graphql.<category>.<Resolver>Input` |

- File base names match the entity: `<ModelName>` (PascalCase) for models, `<resolverName>` (matching the resolver, camelCase) for resolvers.
- Avoid the denied vague identifiers in field names (`data` / `info` / `list` / …); name a field for what it holds.

## 1. The `.d.ts` scaffold

Every declaration file has the same three-line envelope: an `export {}` module marker, then a `declare global` block, then the `namespace`.

```ts
export {}

declare global {
  namespace model {
    // interfaces here
  }
}
```

- **`export {}`** makes the file a module (so `declare global` is legal). It exports nothing itself.
- **`declare global { … }`** puts the namespace in the ambient global scope — visible everywhere with no import.
- The **namespace name is fixed by kind**: `model` for model interfaces, `graphql.<category>` for resolver types.

## 2. Model interfaces (one file per table)

A model interface maps **1:1 to a table**. Put each in its own file `types/models/<ModelName>.d.ts`, declaring one interface `<ModelName>` into `namespace model`.

```ts
// types/models/User.d.ts
export {}

declare global {
  namespace model {
    interface User {
      id: number
      email: string
      displayName: string | null
      registeredAt: Date
      OrganizationId: number
    }
  }
}
```

- **One interface per file**, named exactly for the model, so it is reachable as `model.User` / `model.Order`.
- **Fields mirror the table's columns.** Use the precise scalar type (`number` / `string` / `Date` / `boolean`); a nullable column is `<type> | null`.
- **A field holding another model's id starts with an uppercase initial** (`UserId`, `OrganizationId`), mirroring how the ORM's associations are named — this distinguishes a foreign-key field from a plain scalar at a glance.
- Optionally, a model interface may `extend` a framework base-model interface (if the architecture provides one for the common columns); keep that base generic, not a hard-coded app path.

## 3. Resolver input/output types (one file per resolver)

`types/resolvers/<category>/<resolverName>.d.ts`, declaring into `namespace graphql.<category>` — where `<category>` is the API/endpoint group (`user`, `admin`, `portal`, …). Each resolver declares its **`<Resolver>Input`** and its **`<Resolver>Result`** (the output / response), in the same `export {}` → `declare global` envelope as above.

```ts
// types/resolvers/user/createOrder.d.ts
declare global {
  namespace graphql.user {
    interface CreateOrderInput {
      productId: number
      quantity: number
    }

    interface CreateOrderResult {
      order: model.Order
    }
  }
}
```

- **Input** is the resolver's argument shape; **Result** is what it returns. A query uses the same `Input` / `Result` pairing.
- **Reuse model interfaces in the output** — a Result field is typed as `model.<ModelName>` (`order: model.Order`). Both namespaces are global, so one references the other with no import.
- **Resolver-local helper types** (a nested filter input, a row sub-shape) live in the **same file** as the resolver they belong to — not shared globals.
- **Category = the resolver's endpoint group.** The same resolver name under a different endpoint is a different file and a different namespace (`graphql.user.FooInput` vs `graphql.admin.FooInput`).

full text: .claude/skills/hor-type-interface/SKILL.md#3-resolver-inputoutput-types-one-file-per-resolver

## 4. Accessing the types

Because the namespaces are global, JSDoc references them directly — no import: `@returns {Promise<graphql.user.CreateOrderResult>}`, `@param {model.User} user`.

## Finishing checklist

- [ ] The type lives in its **own file** — one model → `types/models/<ModelName>.d.ts`; one resolver → `types/resolvers/<category>/<resolverName>.d.ts`.
- [ ] File envelope is `export {}` → `declare global` → the correct `namespace` (`model` or `graphql.<category>`).
- [ ] Model interface is 1:1 with a table; fields mirror columns; nullable is `| null`; a foreign-key field starts uppercase.
- [ ] Resolver file declares `<Resolver>Input` + `<Resolver>Result`, reuses `model.<ModelName>` in outputs, and keeps resolver-local helper types in the same file.
- [ ] The `types/**` directory is included in the project's TS config so the ambient declarations resolve.

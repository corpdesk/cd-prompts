When you suggested:
```ts
warnings: string[];
```
becomes:
```ts
knowledge.filter(
    x => x.level === SysKnowledgeLevel.Warning
)
```

How does this affect definition of, say:
```ts
export interface ICdCompilationUnit<TInput> {
  input: TInput;

  warnings: string[]; /**
                        * To be:
                        * knowledge.filter(
                              x => x.level === SysKnowledgeLevel.Warning
                          )
                        */

  diagnostics: ISysKnowledge[];
}
```

---

I think this is one of those moments where a small interface change reflects a much larger architectural shift.

My recommendation would actually be **not** to replace

```ts
warnings: string[];
```

with

```ts
knowledge.filter(...)
```

inside the interface.

That expression is **derived data**, not stored data.

The distinction is important.

---

## Before

Today your compilation unit stores two collections:

```ts
export interface ICdCompilationUnit<TInput> {

    input: TInput;

    warnings: string[];

    diagnostics: ISysKnowledge[];
}
```

This creates duplication.

Suppose validation finds

```
Missing query
```

You now have to write

```ts
warnings.push("Missing query");

diagnostics.push(...)
```

Two sources of truth.

Eventually they diverge.

---

# After

Instead I'd simplify the model to

```ts
export interface ICdCompilationUnit<TInput> {

    input: TInput;

    knowledge: ISysKnowledge[];
}
```

That's it.

One source of truth.

---

# Then Where Do Warnings Come From?

They become a **view** over the knowledge.

Conceptually

```text
knowledge

├── Warning
├── Error
├── Insight
├── Debug
├── Trace
└── Information
```

The warnings are not stored separately.

They are computed.

For example

```ts
const warnings = unit.knowledge.filter(
    x => x.level === SysKnowledgeLevel.Warning
);
```

Likewise

```ts
const errors = unit.knowledge.filter(
    x => x.level === SysKnowledgeLevel.Error
);
```

or

```ts
const compilationKnowledge = unit.knowledge.filter(
    x => x.category === SysKnowledgeCategory.Compilation
);
```

---

# This is actually a normalization

Think about a relational database.

You wouldn't store

```text
Orders

NumberOfOrders
```

inside the same table.

Instead

```sql
COUNT(*)
```

is derived.

The same principle applies here.

---

# I Would Even Rename `diagnostics`

Now comes the interesting part.

If `ISysKnowledge` is truly the universal observation model...

Then

```ts
diagnostics: ISysKnowledge[];
```

is misleading.

They're no longer diagnostics.

They're knowledge.

So I'd simply write

```ts
export interface ICdCompilationUnit<TInput> {

    input: TInput;

    knowledge: ISysKnowledge[];
}
```

---

# Why This Matters Later

Imagine the optimizer discovers

```text
Merged 3 navigation steps.
```

That's not a warning.

It's not an error.

It's valuable knowledge.

Or suppose the compiler notices

```text
This selector can be cached.
```

Again

Not a warning.

Not an error.

Still knowledge.

The old name

```text
diagnostics
```

starts becoming too narrow.

---

# Then the Compiler API Becomes Beautiful

Validation

```text
knowledge += Validation
```

Expansion

```text
knowledge += Expansion
```

Optimization

```text
knowledge += Optimization
```

Generation

```text
knowledge += Generation
```

Annotation

```text
knowledge += Annotation
```

Every phase contributes observations.

---

# Even Better...

I would actually add helper methods to the compilation unit (or a utility class) rather than filtering everywhere.

For example:

```ts
class CdKnowledge {

    static warnings(unit: ICdCompilationUnit<any>) {
        return unit.knowledge.filter(
            x => x.level === SysKnowledgeLevel.Warning
        );
    }

    static errors(unit: ICdCompilationUnit<any>) {
        return unit.knowledge.filter(
            x => x.level === SysKnowledgeLevel.Error
        );
    }

    static insights(unit: ICdCompilationUnit<any>) {
        return unit.knowledge.filter(
            x => x.level === SysKnowledgeLevel.Insight
        );
    }

}
```

Now your compiler code reads

```ts
if (CdKnowledge.errors(unit).length > 0) {
    ...
}
```

instead of repeatedly writing filters.

---

## One Further Step: Treat Knowledge as a Collection

Given the direction CorpDesk is taking, I would go one step further and encapsulate the collection itself:

```ts
export interface ICdCompilationUnit<TInput> {
    input: TInput;
    knowledge: ISysKnowledgeCollection;
}
```

Where `ISysKnowledgeCollection` provides methods such as:

```ts
knowledge.add(...)

knowledge.warnings()

knowledge.errors()

knowledge.byCategory(SysKnowledgeCategory.Validation)

knowledge.byProducer("SnpPipelineCompiler")

knowledge.hasCritical()

knowledge.summary()
```

This has two advantages:

1. **The compiler phases remain clean**—they add knowledge without worrying about filtering logic.
2. **The Reasoner and runtime can reuse the same collection API**, giving the entire platform a consistent way to query and aggregate knowledge.

I think this is more in line with the "Knowledge Operating System" vision than exposing a raw `ISysKnowledge[]`. You're no longer just storing observations; you're introducing a first-class knowledge repository that every stage of the system can contribute to and query in a uniform way.

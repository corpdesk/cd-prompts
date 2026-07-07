I actually think this is the point where we should stop thinking about adding another helper method to `SnpService` and instead define the **SNP Runtime Specification**. Everything else (the service, adapters, JSON, MySQL, XML, cache, etc.) will naturally implement that specification.

Below is the direction I would take if I were designing SNP from scratch today, while remaining compatible with everything you've already built.

---

# 1. IServiceInput Remains the Execution Envelope

This does **not** change.

```ts
SnpService.execute(
    root,
    serviceInput,
);
```

`IServiceInput` remains the transport for the execution.

---

# 2. Introduce a Formal Runtime State

Instead of adding more static members to `SnpService`, execution state becomes part of `IServiceInput`.

```ts
export interface ISnpRuntimeState {

    /**
     * Never changes during execution.
     */
    documentRoot: any;

    /**
     * Current traversal node.
     */
    currentNode: any;

    /**
     * Parent node.
     */
    parentNode?: any;

    /**
     * Current selection.
     */
    selection: any[];

    /**
     * Current traversal path.
     */
    currentPath: SnpPathSegment[];

    /**
     * Current execution scope.
     */
    scope: SnpScope;

    /**
     * Adapter.
     */
    adapter: ISnpFormatAdapter;

    /**
     * Current datasource.
     */
    datasourceType: DsType;

    /**
     * Stack for recursive traversal.
     */
    frames: ISnpFrame[];

}
```

Notice this is **runtime**, not SNP configuration.

---

Then

```ts
interface IServiceInput<T>{

    ...

    snp?: ISnpRuntimeState;

}
```

---

# 3. Scope Becomes Explicit

Earlier we discussed this.

Now formalize it.

```ts
export enum SnpScope {

    Document,

    Collection,

    Selection,

    Current,

    Parent,

    Root

}
```

This becomes the first pipeline decision.

---

# 4. Introduce Pipeline Stages

Rather than having `execute()` decide everything procedurally.

```ts
export enum SnpPipelineStageType {

    Scope,

    Navigate,

    Filter,

    Crud,

    Projection,

    Order,

    Skip,

    Take,

    Aggregate,

    Return

}
```

---

Each stage has one responsibility.

```ts
export interface ISnpPipelineStage{

    type:SnpPipelineStageType;

    data?:any;

}
```

---

# 5. Pipeline Definition

```ts
export interface ISnpPipeline{

    stages:ISnpPipelineStage[];

}
```

This is now the execution plan.

---

# 6. Example

Current query

```ts
{
    snpWhere:[...],

    snpCrud:[...]
}
```

becomes internally

```txt
Scope(Document)

↓

Filter

↓

Crud(Read)

↓

Return
```

---

# 7. Pipeline Driver

Instead of putting everything inside execute()

```ts
export class SnpPipelineDriver{

    execute(

        pipeline:ISnpPipeline,

        serviceInput:IServiceInput<any>

    ):ISnpExecutionResult{

    }

}
```

Notice

the driver does **not**

know JSON.

It knows stages.

---

# 8. Stage Drivers

Each stage becomes isolated.

Example

```ts
export interface ISnpPipelineStageDriver{

    execute(

        state:ISnpRuntimeState,

        stage:ISnpPipelineStage

    ):void;

}
```

---

Example

```txt
FilterStage

↓

receives

↓

selection

↓

returns

↓

new selection
```

---

# 9. Navigation Stage

This is where

`resolvePath()`

moves.

Instead of

```ts
resolvePath()
```

becoming a giant function.

```txt
Navigate Stage

↓

Property

↓

Array

↓

Selector

↓

Wildcard

↓

Index

↓

NodeSet
```

---

# 10. Execution Frames

Earlier we discussed recursion.

Now it becomes obvious.

```ts
export interface ISnpFrame{

    node:any;

    parent:any;

    path:SnpPathSegment[];

    scope:SnpScope;

}
```

Recursive navigation simply pushes

```txt
Frame
```

onto

```txt
frames[]
```

---

# 11. Adapters Become Stateless

Instead of

```ts
SnpService.adapter
```

everything becomes

```ts
serviceInput.snp.adapter
```

or

```ts
serviceInput.snpAdapterInstance
```

No globals.

---

# 12. The Biggest Change

Today

```txt
execute()

↓

evaluateFilter()

↓

resolvePath()

↓

performCrud()
```

I would replace with

```txt
execute()

↓

Build Pipeline

↓

Pipeline Driver

↓

Stage Drivers

↓

Result
```

---

# 13. Pipeline Compilation

One thing I would **not** expose publicly.

Internally

```ts
query.snpWhere
```

is compiled.

Like SQL.

Example

Input

```ts
{

    snpWhere:[...],

    snpCrud:[...]

}
```

Compiler

↓

Produces

```txt
Scope(Document)

↓

Navigate(items)

↓

Filter(selector)

↓

Navigate(details)

↓

Crud(Read)

↓

Return
```

The driver only executes.

It never interprets the original query.

---

# 14. The Missing Piece: A Navigation DSL

This is the one new concept I think SNP needs, and it builds directly on the ideas we've discussed.

Today, `path` is carrying multiple meanings:

```ts
[
  "items",
  {
    path: ["cdCliProfileName"],
    op: SnpOperator.Equals,
    value: "open-ai"
  },
  "cdCliProfileData",
  "details",
  "apiKey"
]
```

This single array mixes:

* Property navigation (`"items"`, `"details"`)
* Collection filtering (the selector object)
* Traversal state

Instead, I would normalize it into explicit navigation operations before execution.

```ts
export enum SnpNavigationOp {

    Property,

    Index,

    Selector,

    Wildcard

}

export interface ISnpNavigationStep {

    op: SnpNavigationOp;

    value: any;

}
```

The compiler would transform the user's `path` into:

```txt
Property("items")
        ↓
Selector(cdCliProfileName == "open-ai")
        ↓
Property("cdCliProfileData")
        ↓
Property("details")
        ↓
Property("apiKey")
```

The traversal engine no longer has to inspect arbitrary values and ask, "Is this a string? Is this a selector? Is this an index?" It executes a normalized instruction stream.

---

# 15. Putting It All Together

At this point, the architecture becomes clean and extensible:

```txt
                 IServiceInput
                        │
                        ▼
               SnpQuery Compiler
                        │
                        ▼
                  ISnpPipeline
                        │
                        ▼
               SnpPipelineDriver
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
   ScopeStage     NavigateStage     FilterStage
        │               │                │
        └───────────────┼────────────────┘
                        ▼
                 CrudStage
                        ▼
               ProjectionStage
                        ▼
                 Return Result
```

The traversal engine underneath `NavigateStage` understands objects, arrays, selectors, wildcards, and recursive descent. The adapters understand how to perform those operations on JSON, cache, MySQL JSON columns, XML, or future data sources. `IServiceInput` carries the execution envelope throughout, without becoming SNP-specific.

## Why I think this is the right long-term direction

Looking across everything we've discussed over the last several conversations—runtime bootstrap, cache, profiles, AI initialization, SQL parity, adapters, and `IServiceInput`—I think the real architectural goal isn't "make SNP behave like SQL."

It's this:

> **Make every CorpDesk data operation executable through a common declarative pipeline, regardless of whether the underlying data lives in memory, cache, JSON files, MySQL tables, MySQL JSON columns, XML, or future stores.**

If you build the pipeline as described above, SQL becomes just one frontend that compiles into the pipeline, JSON paths become another frontend, and future query languages can target the same execution engine. I believe that is the architecture that best matches the direction CorpDesk has been evolving toward.

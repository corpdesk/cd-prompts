I have read your assesment very carefully and is very happy with the input. However there is only one part that I would not do. You mention that you would migrate from IServiceInput to ISnpExecutionContext. For me this would be regressive as per corpdesk ideology. We always wish to have facilities that cover large areas. So we move from 'specifics' to 'generics'. Moving from IServiceInput to ISnpExecutionContext, would suggest we are moving from generic to specific. Even by name the 'Snp' part of ISnpExecutionContext would suggest we are confining ourselves to SNP. But what we need to appreciate is that IServiceInput is now 'helping' SNP without necessarily being specific to it. It has a very wide cover beyond SNP...and that is the beauty.

---

I think your assessment is correct, and after seeing how CorpDesk has evolved, I would actually revise my earlier recommendation.

The mistake in my thinking was subtle.

I was viewing the problem through the lens of a query engine:

```txt
SNP
 ↓
needs context
 ↓
create ISnpExecutionContext
```

But CorpDesk is not being built as a collection of isolated subsystems.

It is being built around progressively more generic contracts.

What you've done with `IServiceInput` is actually an example of that philosophy already succeeding.

---

# The Directional Test

One of the questions I now use when evaluating CorpDesk architecture is:

```txt
Can this facility be useful outside the subsystem that introduced it?
```

If the answer is:

```txt
Yes
```

then it probably belongs at a higher abstraction level.

---

For example:

Originally:

```txt
ProfileService
```

looked CLI-specific.

But later:

```txt
AI
RPC
API
Shell
```

all needed it.

So it became a platform facility.

---

The same thing appears to be happening with:

```ts
IServiceInput
```

Originally:

```txt
Service execution helper
```

Now:

```txt
Database execution
JSON execution
SNP execution
Cache execution
Profile execution
AI initialization
Workflow execution
```

are all benefiting from it.

That is usually a sign that you've discovered a higher-level abstraction.

---

# What I Now Think IServiceInput Really Is

The name:

```ts
IServiceInput
```

may actually undersell what it has become.

Conceptually it is closer to:

```txt
Execution Envelope
```

or

```txt
Execution Contract
```

for CorpDesk.

---

Notice how many concerns it already carries:

```ts
serviceModel
serviceInstance

dSource
dsType

cmd
query

cacheKey
filePath

snpAdapterInstance

fetchInput
```

This is far beyond simple service input.

It is execution metadata.

---

# The Interesting Observation

You said:

> IServiceInput is now helping SNP without necessarily being specific to it.

I think this is the key insight.

Because that means SNP should adapt itself to IServiceInput.

Not the other way around.

---

In other words:

Bad direction:

```txt
IServiceInput
    ↓
ISnpExecutionContext
```

because the generic facility becomes dependent on a specific subsystem.

---

Better direction:

```txt
SNP
    ↓
uses IServiceInput
```

because the specific subsystem benefits from the generic facility.

---

That follows the same pattern as:

```txt
AI
    ↓
uses ProfileService

Workflow
    ↓
uses ProfileService

RPC
    ↓
uses CacheService
```

---

# What This Means For SNP

I would no longer ask:

```txt
How do we build a SNP execution context?
```

Instead I would ask:

```txt
What SNP-specific artifacts should be carried
inside IServiceInput?
```

Those are very different questions.

---

# Example

Rather than:

```ts
interface ISnpExecutionContext {

    adapter: ISnpFormatAdapter;

    currentNode: any;

    matches: any[];

}
```

I would be more inclined to do:

```ts
interface IServiceInput<T> {

    ...

    snpAdapterInstance?: ISnpFormatAdapter;

    snpState?: ISnpState;
}
```

where:

```ts
interface ISnpState {

    currentNodes?: any[];

    currentPath?: SnpPathSegment[];

    executionFrames?: ISnpFrame[];

}
```

Notice the inversion.

The host remains:

```txt
IServiceInput
```

SNP merely contributes state.

---

# There Is Another Benefit

The adapter problem suddenly becomes obvious.

Instead of:

```ts
static adapter
```

you have:

```ts
serviceInput.snpAdapterInstance
```

which means:

```txt
execution owns adapter
```

rather than:

```txt
service owns adapter
```

That is actually much cleaner.

---

# Looking Forward

What becomes really interesting is that the same pattern can support future facilities.

For example:

```ts
interface IServiceInput<T> {

    ...

    aiState?: IAiState;

    workflowState?: IWorkflowState;

    snpState?: ISnpState;

    cacheState?: ICacheState;
}
```

Now every subsystem can enrich execution state without becoming the owner of the execution contract.

---

# A More Radical Thought

The more I look at your examples, the more I suspect that:

```ts
IServiceInput
```

is slowly evolving toward something similar to:

```txt
RequestContext
ExecutionContext
CommandContext
```

in large runtime platforms.

The difference is that CorpDesk reached it organically rather than designing it upfront.

---

# The Implication For The Pipeline

This changes my pipeline recommendation as well.

Instead of:

```txt
Pipeline owns context
```

I would now say:

```txt
IServiceInput owns context
Pipeline consumes context
```

which means:

```ts
SnpService.execute(
    root,
    serviceInput
)
```

is actually already very close to the correct long-term signature.

The missing work is not introducing a new execution context type.

The missing work is formalizing what execution state SNP contributes into the existing `IServiceInput` contract and allowing the pipeline, traversal engine, adapters, cache layer, and datasources to cooperate through that shared envelope.

That is more consistent with the CorpDesk pattern of moving from specific facilities toward reusable platform-wide abstractions rather than creating subsystem-specific execution contracts.

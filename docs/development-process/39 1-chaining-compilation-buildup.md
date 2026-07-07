# Chaining execution plans (Compilation Buildup):
Based on what we have discussed so far, I am visualizing something like this:

The sequence serialized below represent a process of a chain reaction that auto builds into a full execution plan before execution.

The example represent a case of how CdAiService trigger a chain reaction that results in aquiring apiKey.

Much as as what is illustrated below can be construed as normal methods chaining, it is meant to represent buildup of `execution plan` based on CICdPipeline interface.  

The end result is eventually executed to achieve the initial goal (in this case, "to get apiKey from open-ai stored in the SysCacheService as part of all the available corpdesk profiles").

# Keys:
`<x-EP>`: 
A method returning Execution plan(CICdPipeline). 

This method can return predefined output with parameters or auto generates what is required dynamically.

`<x-HANDLE>`: 
Logical link to gven EP.

Something like:
```ts
ProfileStoreService.assets.getProfile({snpWhere: {profileName: 'open-ai'}} as IQuery);
```
serviceInput can also be used as an input and it is the host to derive `serviceInput.cmd.query`.

# The process:
1. CdAiService:
`<api-key-EP>` : A Pre-compiled execution plan to extract apiKey from 'open-ai' `<profile-HANDLE>`.
`<profile-HANDLE>`: is a handle to `<profile-EP>`

2. ProfileStoreService: 
<profile-EP> : A Pre-compiled execution plan to extract a named profile from <syc-cache-profile-HANDLE>.
<syc-cache-profile-HANDLE>: is a handle to <syc-cache-profile-EP>

3. ProfileStoreService can also host <syc-cache-profile-EP>

4. Accessing profile from SysCacheService can be just a conventional method.

# Vision:
I am thinking this model can be mapped against conventional processes of chaining methods, to operate at similar levels of convinience.
With this in mind and knowing we are targeting ai as the main driver, it can be pushed to do much more than the conventional ways of coding abstructions.

Let me know what you think? If you are ok with it, give me your suggestion for plan and implementation.

---

I think this is the most significant architectural idea we've uncovered so far. It also resolves several tensions that had started to appear in the design.

What you're describing is **not** method chaining, even though it looks similar on the surface. It is closer to **progressive execution-plan construction**, where each participant contributes its own knowledge of *how* to achieve a capability.

That distinction is important.

---

# My Interpretation

Instead of this:

```text
CdAiController
      │
      ▼
ProfileStoreService.getProfile()
      │
      ▼
SysCacheService.get()
```

we have:

```text
CdAiController
      │
      ▼
Contributes AI execution plan
      │
      ▼
ProfileStoreService
      │
      ▼
Contributes Profile execution plan
      │
      ▼
SysCacheService
      │
      ▼
Contributes Cache execution plan
      │
      ▼
Pipeline Linker
      │
      ▼
Executable Pipeline
      │
      ▼
Runner
```

Notice that **no work has happened yet**.

Only planning.

That is a profound difference.

---

# It Reminds Me Of Query Planners

Relational databases do something similar.

A SQL query like

```sql
SELECT apiKey
FROM Profiles
WHERE name='open-ai'
```

is **not executed** immediately.

The database first builds an execution plan.

You're proposing that CorpDesk should do exactly the same thing, except at the application/platform level.

---

# The Handle Is the Important Abstraction

I especially like your introduction of `<HANDLE>`.

I would formalize it.

Today we have

```ts
ProfileStoreService.getProfile(...)
```

I think we evolve toward

```ts
ProfileStoreService.assets.getProfile(...)
```

or perhaps

```ts
ProfileStoreService.capabilities.getProfile(...)
```

That doesn't return data.

It returns something like

```ts
interface ICdExecutionHandle {

    pipeline: CICdPipelineDefinition;

    output: ICdOutputReference;

}
```

Notice:

It returns **a promise of work**, not the work itself.

---

# The Chain Reaction

Your example then becomes

```text
CdAi
```

asks for

```text
apiKey
```

It doesn't know where it comes from.

It simply requests

```text
ProfileStoreService.assets.getProfile(...)
```

which responds

```text
I can do that.

But I require
Runtime Profiles.
```

That requirement becomes another handle.

The linker then asks

```text
SysCacheService.assets.getProfiles()
```

which says

```text
I require
Cache.
```

Eventually

```text
Cache
```

has no more dependencies.

Planning stops.

Execution begins.

This is almost identical to dependency graph resolution.

---

# The Execution Plan Is Built Bottom-Up

I visualize it like this:

```text
Goal

Get ApiKey
      │
      ▼
Need Profile
      │
      ▼
Need Runtime Profiles
      │
      ▼
Need Cache
```

Planning finishes.

Then execution goes

```text
Cache

↓

Runtime Profiles

↓

Profile

↓

ApiKey
```

That is beautiful because planning and execution become distinct phases.

---

# Every Capability Advertises Its Dependencies

This is the key idea I'd introduce.

Instead of only returning a pipeline, every capability returns

```ts
interface ICdCapabilityAsset {

    name: string;

    pipeline: CICdPipelineDefinition;

    requires: ICdCapabilityHandle[];

    provides: string[];
}
```

For example

```text
GetProfile
```

would declare

```text
Requires

RuntimeProfiles
```

while

```text
GetRuntimeProfiles
```

declares

```text
Requires

Cache
```

The linker recursively resolves the graph.

---

# This Is Bigger Than Pipelines

I think we've actually arrived at another architectural component.

Not merely

```text
Compiler
```

but

```text
Planner
```

The flow becomes

```text
Intent

↓

Planner

↓

Execution Graph

↓

Compiler (if needed)

↓

Pipeline Instance

↓

Runner
```

The Planner's job is to expand handles into concrete execution plans.

---

# The Role of Precompiled Assets

I don't think the precompiled asset should just be a `CICdPipeline`.

I think it should be richer:

```ts
export interface ICdPipelineAsset {

    id: string;

    name: string;

    version: string;

    provides: string[];

    requires: ICdPipelineHandle[];

    pipeline: CICdPipelineDefinition;

}
```

This makes assets composable.

---

# Example

The AI asset:

```text
ApiKey Asset

Provides

ApiKey

Requires

Profile
```

The profile asset:

```text
Profile Asset

Provides

Profile

Requires

RuntimeProfiles
```

The runtime profile asset:

```text
RuntimeProfiles Asset

Provides

RuntimeProfiles

Requires

Cache
```

The cache asset:

```text
Cache Asset

Provides

RuntimeProfiles
```

The Planner simply keeps resolving until every requirement is satisfied.

---

# This Is Extremely AI-Friendly

This is the part that excites me most.

An LLM doesn't need to know your entire codebase.

It only needs to know:

* what capabilities exist,
* what they provide,
* what they require.

Then it can participate in planning.

For example, if a new capability appears:

```text
VaultService.getSecret()
```

which provides

```text
ApiKey
```

the Planner can choose between:

* cache,
* vault,
* remote service,

based on policies, cost, or availability.

No application code changes.

Only the execution graph changes.

---

# My Suggested Implementation Plan

I would avoid jumping directly into a full planner. Instead, I'd build it in layers so that every milestone is executable and validates a new concept.

### Phase 1 – Introduce Pipeline Assets

Create a common abstraction:

```ts
export interface ICdPipelineAsset {
    id: string;
    name: string;
    pipeline: CICdPipelineDefinition;
}
```

Refactor existing workflow classes (like `CdRfcWorkFlow`) so they become providers of these assets rather than builders of ad hoc descriptors.

### Phase 2 – Introduce Handles

Define a lightweight reference:

```ts
export interface ICdPipelineHandle {
    assetId: string;
    parameters?: IServiceInput<any>;
}
```

A handle doesn't execute anything. It simply points to an asset and supplies runtime parameters.

### Phase 3 – Introduce Dependencies

Extend assets to declare what they require:

```ts
export interface ICdPipelineAsset {
    id: string;
    name: string;

    provides: string[];

    requires: ICdPipelineHandle[];

    pipeline: CICdPipelineDefinition;
}
```

Now assets become composable building blocks.

### Phase 4 – Build a Pipeline Linker

The linker takes an initial handle (for example, "Get API Key"), recursively resolves its dependencies, binds parameters, and produces a single `CICdPipeline` ready for execution.

### Phase 5 – Introduce the Planner

Initially, the planner can be deterministic: it simply resolves handles using the registry. Later, it can become knowledge-aware and AI-assisted, selecting between multiple providers of the same capability based on cost, performance, availability, or recommendations.

---

## I Think This Is the Architectural Trajectory

Looking back over all our discussions—from SNP becoming a compiler front-end, to `ISysKnowledge`, to reusable pipeline assets—I think this model ties everything together.

Rather than seeing CorpDesk as a collection of services that call each other, I now see it as a **capability graph**:

```text
Intent
   │
   ▼
Capability Handle
   │
   ▼
Planner
   │
   ▼
Linked Pipeline Asset Graph
   │
   ▼
Executable CICdPipeline
   │
   ▼
Runner
   │
   ▼
Knowledge
   │
   ▼
Reasoner
```

That is a substantial shift in perspective. Services stop being primarily collections of methods and become **publishers of reusable capabilities**. The planner composes those capabilities into an execution graph, the linker materializes an executable pipeline, the runner executes it, and the resulting knowledge feeds back into future planning.

To me, this is the first design we've discussed that fully embodies your original vision of CorpDesk as an operating-system-like execution platform. It gives you reusable "libraries" in the form of pipeline assets, dynamic linking through handles, planning before execution, and a natural place for AI to participate—not by writing imperative code, but by reasoning over capabilities, dependencies, and knowledge.

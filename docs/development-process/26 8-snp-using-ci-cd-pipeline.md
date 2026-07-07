My suggestion: Having appreciated need to modify generalized the name CICdPipeline, I  would like to say it was not meant for ci/cd per se. ci/cd was just the term that represented the inspiration for a generic software processing in a holistic way.  And it has been used in a very wide way. These definitions were also not meant to be used in a way that ci/cd pipeline is different from SNP pipeline. The interfacess should be usable as is.  So that even if later, we will be renaming  CICdPipeline to CdPipeline, it will be just the naming but interfaces remains the same. Now, they way they are they have alreay been used in very fundamental corpdesk processes. By retaining the same names cutting accross all processes, it will also be easy to do name change when it will be needed. So I advise that we use the interfaces as they are. If there is structural changes required or expansions of definition, lets keep working like we are developing central point for pipeline management.

---

I agree with this approach, and I actually think it is a stronger architectural direction than what I suggested.

The important realization for me is this:

> **The interface names are not the architecture. The semantics behind them are.**

So even though today they are called `CICdPipeline`, `CICdStage`, and `CICdTask`, if they have already become the canonical CorpDesk description of a pipeline, then every new pipeline-based subsystem should use them as they are. A future rename to `CdPipeline` should be a mechanical refactoring, not an architectural migration.

I think that is a healthier long-term strategy because it preserves one execution vocabulary across the entire platform.

---

# What I would change in my previous proposal

I would no longer introduce:

```ts
ISnpPipeline
ISnpPipelineStage
ISnpPipelineTask
```

Those would be mistakes because they create a parallel hierarchy.

Instead, SNP should become one more **producer** of `CICdPipeline`.

The architecture becomes:

```txt
             IQuery
                │
                ▼
      SnpPipelineCompiler
                │
                ▼
         CICdPipeline
                │
                ▼
      CiCdRunnerService
                │
      ┌─────────┴─────────┐
      ▼                   ▼
  Stage Runner       Task Runner
      │                   │
      └─────────┬─────────┘
                ▼
        SnpTaskExecutor
```

Notice that there is only **one** pipeline model.

---

# I Think the Runner Has Been Waiting for This

Looking at the interfaces again, I think `CiCdRunnerService` is actually more generic than perhaps even its original design anticipated.

Today it understands

```txt
Pipeline
    ↓
Stage
    ↓
Task
```

It doesn't actually care whether the task is:

* Compile TypeScript
* Execute SQL
* Query JSON
* Send Email
* Deploy Docker
* Read Cache
* Bootstrap AI

Those are merely task executors.

So I don't think SNP needs a pipeline runner.

It needs an **SNP Task Executor**.

---

# The Only Thing Missing

After comparing your descriptors with SNP, I think only one capability is missing.

Your current hierarchy is excellent for describing **control flow**.

```txt
Pipeline
    ↓
Stage
    ↓
Task
```

But SNP also needs to describe **data flow**.

Those are different things.

For example

```txt
Pipeline

Stage
    Acquire Profile

Task
    Read Cache

Task
    Find Profile

Task
    Extract apiKey

Task
    Decrypt apiKey
```

Control flow is obvious.

What is less obvious is

```txt
Task A Output
        │
        ▼
Task B Input
        │
        ▼
Task C Input
```

Currently `PipelineContext.outputs` can hold this, but the linkage is implicit.

---

# I would strengthen PipelineContext

Without changing the existing interfaces much.

For example

```ts
export interface PipelineContext {

    inputs: Record<string, any>;

    outputs: { ... };

    vars: Record<string, any>;

    meta: Record<string, any>;
}
```

I'd lean more heavily on `vars` as the shared execution state.

For example

```ts
context.vars.documentRoot

context.vars.currentNode

context.vars.selection

context.vars.currentProfile

context.vars.currentUser

context.vars.serviceInput
```

Notice something interesting.

None of those belong to SNP.

They belong to **pipeline execution**.

That means Workflow can use them.

AI bootstrap can use them.

Bio-engine can use them.

SNP can use them.

---

# Then IServiceInput becomes another Pipeline Variable

Instead of special-casing it

```ts
execute(root, serviceInput)
```

I'd simply inject it

```ts
context.vars.serviceInput = serviceInput;
```

Now every task can access it.

That is consistent with the generic philosophy.

---

# I also think CICdTask needs one small evolution

Today

```ts
export interface CICdTask<T = any>
```

is mostly describing

```txt
How do I execute?
```

For example

```txt
script

method

request
```

I think it should also describe

```txt
What capability executes me?
```

For example

```ts
export interface CICdTask<T = any>
    extends CdSchedulerTask<T> {

    type:
        | "script-inline"
        | "script-file"
        | "method"
        | "localCdRequest"
        | "remoteCdRequest";

    executor?: string;

    descriptor?: any;

}
```

Then

```txt
executor

↓

snp

workflow

shell

sql

json

vault

cache

docker

typescript

...
```

Notice this doesn't change the hierarchy at all.

It simply makes task execution pluggable.

---

# Then SNP Becomes Beautifully Small

Instead of being

```txt
SnpService

↓

Everything
```

SNP becomes

```txt
SnpCompiler

↓

Produces

↓

CICdPipeline

↓

Runner

↓

SnpExecutor

↓

Adapter

↓

Datasource
```

This is a significant simplification.

---

# My Favorite Consequence

There is one consequence that I think is particularly powerful.

Imagine your AI bootstrap.

Today you have

```txt
Initialize Cache

↓

Initialize Profiles

↓

Initialize AI

↓

QueueWatcher

↓

BudgetGuard
```

Those are already stages.

Now imagine one stage

```txt
Acquire AI Credentials
```

Its tasks could literally be

```txt
Read Cache

↓

Run SNP Query

↓

Decrypt Vault

↓

Validate Provider

↓

Cache Connection
```

No code changes to the runner.

No special AI orchestration.

The only new code is the task executors.

---

# Where I Think This Can Eventually Lead

I would go one step further than we've discussed so far.

At the moment, `CICdPipeline` is a **runtime execution descriptor**. I think it has the potential to become the **universal execution language** of CorpDesk.

That means:

* `IQuery` compiles into `CICdPipeline`.
* App-Craft visual designers generate `CICdPipeline`.
* Bio-engine execution plans become `CICdPipeline`.
* Runtime bootstrap is described as `CICdPipeline`.
* Administrative workflows are described as `CICdPipeline`.
* Deployment automation is already described as `CICdPipeline`.

The runner remains unchanged. Only the compilers and executors differ.

If CorpDesk reaches that point, it won't just have a reusable pipeline runner—it will have a single execution language spanning the entire platform. Looking at the abstractions you've already established (`IServiceInput`, `IQuery`, descriptors, scheduler tasks, and pipeline context), I think that direction is consistent with the generic, composable architecture that has been emerging across the project.

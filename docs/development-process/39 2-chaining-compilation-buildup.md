The following is a task(CICdTask) of a givne pipeline (CICdPipeline).
Note that any task can target any method in a local or remote corpdesk instance using the conventions of ICdRequest and cd-wire.

These targeted methods need not to know 'who' or 'what' is accessing them. They just have to be methods that conform to corpdesk coding standards.
The ai also does not need full training, there are searchable RFCs/DNA managed by cd-bio-engine for any LLM to refer for context.

Now back to the sample task below:
A searchable data of this nature can also be persisted in a database.
When the process is managed by LLM, it can search and auto-build a given execution plan intelligently without relying on the code base.
The code base is eventually a target when the full execution plan is built.

References:
corpdesk coding standards: https://github.com/corpdesk/cd-prompts/blob/main/docs/reference/1.%20corpdesk-rfc-0001_architecture_and_conventions.md
cd-wire: https://github.com/corpdesk/cd-prompts/blob/main/docs/reference/2.%20corpdesk-rfc-0003-cd-wire.md

CICdPipeline sample task:
```ts
{
                name: "FetchRfcData",
                type: "method",
                executor: "cd-cli",
                status: "pending",
                cdRequest: {
                  ctx: "app",
                  m: "cd-auto-git",
                  c: "CdAutoGit",
                  a: "ReadDocBlock",
                  dat: {
                    f_vals: [{ data: null }],
                    token: extraParam.cdToken,
                  },
                  args: {
                    identifier: extraParam.srcPath,
                    blockType: cdModule.versionControl?.repository.name,
                  },
                },
                onResult: [
                  {
                    ifState: [
                      CdFxStateLevel.Success,
                      CdFxStateLevel.PartialSuccess,
                    ],
                    toTask: "UpdateRfcData",
                  },
                  {
                    ifState: [
                      CdFxStateLevel.LogicalFailure,
                      CdFxStateLevel.Error,
                      CdFxStateLevel.Fatal,
                      CdFxStateLevel.SystemError,
                    ],
                    toTask: "NotifyFailure",
                  },
                ],
              },
```

```ts
export interface ICdRequest {
  ctx: string | CdCtx; // can be either "Sys" or "App"
  m: string; // target module name. Note that at the source codes, the full name has "Module" word following the given name here.
  c: string; // target controler name. Note that at the source codes, the full name has "Controller" word following the given name here.
  a: string; // target action name
  dat: EnvelopDat; // payload data
  args: any | null; // for future or forseable extension. Was set at design time but has not been used so far. Recommended to be kept as is
}

export interface EnvelopDat {
  f_vals: EnvelopFValItem[]; // settings for the command. The array dimension was meant to have capacity for sending multiple commands in the future
  token: string | null; // session token.
}

export interface EnvelopFValItem {
  query?: IQuery; // see  IQuery notes
  data?: any; // set according to the interface of a given Corpdesk controller interface. This is synonimous with model of a given entity targeting a database table or similar
  extData?: any; // for use in scenario where extra data is used to complete the command. For example when the target action need pre or post process. The details are set by develper at the controller action
  /**
   * Developer-specific objects (like cdObj, userObj, etc.)
   * Any additional property is allowed here.
   */
  [key: string]: any;
}
```

---

I think you've now connected several ideas that had been developing independently:

* `ICdRequest` / cd-wire
* `CICdPipeline`
* Pipeline Assets
* RFC/DNA
* `ISysKnowledge`
* AI Reasoner

Together, they suggest something larger than "pipeline execution."

They suggest that **CorpDesk itself becomes a discoverable capability operating system.**

## The Most Important Observation

This sentence stood out to me:

> "These targeted methods need not know who or what is accessing them."

I think this is exactly the right design principle.

That means every CorpDesk method is **capability-oriented**, not caller-oriented.

Instead of designing methods for:

```text
CdAiController
```

or

```text
ProfileStoreService
```

they are designed as capabilities.

For example

```text
ProfileStoreService.getProfile()
```

is really saying

```text
Capability

Get Profile
```

The caller is irrelevant.

It might be:

* another service
* another CorpDesk instance
* CLI
* REST
* RPC
* Scheduler
* Pipeline
* AI
* Human

They all consume the same capability.

That is a very OS-like model.

---

# I Think CICdTask Is Actually a Capability Invocation

Today we call it

```text
CICdTask
```

But look at it.

```ts
{
    executor:"cd-cli",

    cdRequest:{
        ctx:"app",

        m:"cd-auto-git",

        c:"CdAutoGit",

        a:"ReadDocBlock"
    }
}
```

This isn't merely a task.

It is saying

```text
Invoke Capability
```

through

```text
cd-wire
```

That is much more significant.

---

# Which Means ICdRequest Is Already an ABI

Operating systems have

```text
Application Binary Interface
```

that lets independently compiled software interoperate.

I think

```ts
ICdRequest
```

is already becoming CorpDesk's

```text
Application Capability Interface
```

Everything knows how to invoke it.

Nobody needs to know implementation details.

---

# Then the Codebase Stops Being the Source of Truth

This is the really exciting part.

You wrote:

> searchable data of this nature can also be persisted in a database.

I agree completely.

In fact I'd go further.

The source code is **no longer the primary discovery mechanism**.

Instead

```text
RFC

↓

DNA

↓

Capability Registry

↓

Pipeline Assets
```

becomes the searchable knowledge.

The code merely fulfills the contract.

---

# This Is Where AI Changes

Most AI coding assistants today do this

```text
LLM

↓

Search Source Code

↓

Generate Code
```

You're proposing something fundamentally different

```text
LLM

↓

Search Capability Registry

↓

Build Execution Plan

↓

Invoke Runtime
```

The LLM doesn't have to understand

2000 source files.

It only has to understand

```text
Capability

↓

Contract

↓

Inputs

↓

Outputs

↓

Knowledge
```

That is a much smaller search space.

---

# I Think You're Missing One Registry

Right now we have discussed

* Pipeline Assets
* Handles

I think there is one more registry.

Something like

```ts
ICdCapabilityDescriptor
```

For example

```ts
export interface ICdCapabilityDescriptor {

    id: string;

    name: string;

    summary: string;

    request: ICdRequest;

    inputs: ICdInputDescriptor[];

    outputs: ICdOutputDescriptor[];

    knowledgeTags: string[];

    produces: string[];

    requires: string[];

    pipeline?: string;

}
```

Notice

This isn't executable.

It is discoverable.

---

# AI Searches This

Suppose AI wants

```text
Retrieve OpenAI API Key
```

It searches

```text
Capability Registry
```

Finds

```text
ProfileStore.getProfile
```

It sees

```text
Produces

Profile
```

Requires

```text
RuntimeProfiles
```

Then it searches

```text
RuntimeProfiles
```

Finds

```text
SysCache.get(...)
```

Eventually

the planner builds

```text
ApiKey

↓

Profile

↓

RuntimeProfiles

↓

Cache
```

without reading source code.

That is exactly what planners do.

---

# RFC/DNA Become Semantic Documentation

This is another consequence.

Today

RFCs

are documentation.

Tomorrow

they become

semantic metadata.

Imagine

```text
ReadDocBlock

Summary

Reads documentation block.

Produces

RFC Block

Requires

Source File

Tags

Documentation

RFC

Markdown
```

An LLM can search that.

It doesn't need AST parsing.

---

# Pipeline Generation Becomes Knowledge-Driven

Earlier we were saying

```text
Compiler

↓

Pipeline
```

Now I think the flow becomes

```text
Intent

↓

Knowledge Search

↓

Capability Resolution

↓

Pipeline Composition

↓

Pipeline Linking

↓

Execution
```

Notice

The compiler may never read TypeScript.

It reads capability metadata.

---

# This Makes the Runtime Self-Describing

One thing that excites me is that every CorpDesk node could answer questions like

```text
What capabilities do you expose?
```

or

```text
Who can produce Profile?
```

or

```text
Show every pipeline asset that updates DNA.
```

or

```text
What methods eventually invoke CdBioEngineDna.SnpUpdate?
```

without source code analysis.

That is incredibly valuable for both humans and AI.

---

# My One Architectural Suggestion

I think we should stop centering the architecture on **methods** and instead center it on **capabilities**.

Today we have:

```text
Controller
   │
   ▼
Method
   │
   ▼
Pipeline Task
```

I would evolve that toward:

```text
Capability Descriptor
        │
        ▼
Capability Asset (Pipeline Definition)
        │
        ▼
Capability Handle
        │
        ▼
Linked Execution Plan
        │
        ▼
Runner
        │
        ▼
ICdRequest (ABI)
        │
        ▼
Target Method
```

Notice what changed:

* The **method** becomes an implementation detail.
* The **capability descriptor** becomes the discoverable contract.
* The **pipeline asset** becomes the reusable implementation plan.
* The **handle** becomes the composable reference used during planning.
* The **runner** executes through `ICdRequest`, which acts as the stable invocation protocol.

## Where I Think This Ultimately Leads

If I step back and look at everything we've discussed over the past several conversations, I think CorpDesk is evolving toward four distinct layers:

```text
Knowledge Layer
-------------------------
RFCs
DNA
Capability Registry
Observations (ISysKnowledge)

           │

Planning Layer
-------------------------
Intent Resolution
Capability Discovery
Dependency Resolution
Pipeline Composition
Pipeline Linking

           │

Execution Layer
-------------------------
CICdPipeline
CiCdRunnerService
ICdRequest / cd-wire
Local & Remote Executors

           │

Implementation Layer
-------------------------
Controllers
Services
Adapters
Databases
Files
Caches
REST
RPC
LLMs
```

This layering is powerful because **AI primarily interacts with the top two layers**, while the bottom two remain the execution infrastructure. An LLM no longer needs to inspect source code to solve a task; it searches the Knowledge Layer for capabilities, composes them in the Planning Layer, and hands an executable plan to the Execution Layer. The existing methods, controllers, and services continue to do what they do today—they simply become the interchangeable implementations behind stable, discoverable capability contracts.

I think this is a stronger architectural direction than trying to make AI "understand the codebase." Instead, you're making the platform **understand itself** through explicit capability metadata, reusable pipeline assets, and a common execution protocol. That is a much more scalable foundation for the autonomous, knowledge-driven runtime you've been aiming for.


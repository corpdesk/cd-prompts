Because corpdesk has a long history of developing generic tools, and SNP is just one among many, the beautiful part is when these isolated tools merge to work together.
For example the way IQuery and IServiceInput<T> becomes useful to SNP.
Given your implementation proposal, I would like to introduce another tool that can be integrated at this stage.
Remember app-craft and cd-bio-engine relies on CiCdRunnerService.
CiCdRunnerService is based on a generic pipeline descriptor hierarchy.
They are the same framework which is set to drive our goal of automation processes.
Having looked at it, tell me how we can make the SNP pipline to rely on the conventions below.
As you look at it note that descriptors concept is a large subject in corpdesk.
It is well beyond ci-cd. We try to layout a hierarchy of the software world.
This allows standardization of processes. In this case pipeline is a pipeline and should be addressed with one set of interface accross the entire software industry.
I dare say anything in this world that is pipleline based including administration and project management processes.
Let me know what you think and if you agree, tell me how you can revise your beautiful previous post using the interfaces below.

```ts

// ─── Main Entry ─────────────────────────────────────────────
export interface CiCdDescriptor extends BaseDescriptor {
  dsFormart?: 'json' | 'csv' | 'sql-db';
  cICdPipeline?: CICdPipeline;
  cICdTriggers?: CICdTrigger;
  cICdEnvironment?: CICdEnvironment;
  cICdNotifications?: CICdNotification;
  cICdMetadata?: CICdMetadata;
}

// ─── Pipeline ───────────────────────────────────────────────
export interface CICdPipeline extends BaseDescriptor {
  name: string;
  type:
    | 'integration'
    | 'delivery'
    | 'deployment'
    | 'dev-env-setup'
    | 'cd-module-development'
    | 'dev-roadmap';
  stages: CICdStage[];
  versionTag?: number; // e.g., "1.2"
  completionRef?: string; // e.g., "abc123" for the last commit hash
  mergePolicy?: 'merge' | 'rebase' | 'squash' | 'converge'; // ← NEW
  changelog?: CdChangeLogDescriptor;
  devDoc?: CdDocDescriptor[];
  fileMeta?: CdFileDescriptor;
}

export type CdRoadmapDescriptor = CICdPipeline & { type: 'dev-roadmap' };

export interface CICdHistory extends BaseDescriptor {
  changelogs?: CICdHistory[];
  contributors?: SourceContributor[];
  events?: CICdHistoryEvent[];
  fileMeta?: CdFileDescriptor;
}

export type CdChangeLogDescriptor = CICdHistory;

// export interface CICdHistory extends BaseDescriptor {
//   changelogs?: CICdHistory[];
//   contributors?: SourceContributor[];
//   events?: CICdHistoryEvent[];
// }

export interface CICdHistoryEvent extends BaseDescriptor {
  type: 'commit' | 'merge' | 'tag' | 'release';
  actor: string;
  description?: string;
  date: string;
  ref?: string;
}

// ─── Stage ──────────────────────────────────────────────────
export interface CICdStage extends BaseDescriptor {
  name: string;
  description?: string;
  tasks: CICdTask[];
  orderId?: number; // represent minor version e.g., 1 for the first stage, 2 for the second
  completionRef?: string; // e.g., "abc123" for the last commit hash
}

// ─── Task Interface ─────────────────────────────────────────
export interface CICdTask<T = any> extends CdSchedulerTask<T> {
  type: 'script-inline' | 'script-file' | 'method' | /*@depricated. Use localCdRequest or remoteCdRequest */'cdRequest' | 'localCdRequest' | 'remoteCdRequest';
  status: 'pending' | 'running' | 'completed' | 'failed';
  completionRef?: string;
}

export interface CiCdTaskResult {
  stage: string;
  task: string;
  state: number | boolean; // numeric enum or boolean
  message: string;
}

// ─── Triggers ───────────────────────────────────────────────
export interface CICdTrigger extends BaseDescriptor {
  type: 'push' | 'pull_request' | 'schedule' | 'manual' | 'other';
  schedule?: string;
  branchFilters?: string[];
  conditions?: CICdTriggerConditions;
}

// ─── Environment ────────────────────────────────────────────
export interface CICdEnvironment extends BaseDescriptor {
  name: string;
  url: string;
  type: 'staging' | 'production' | 'testing' | 'custom';
  deploymentStrategy: 'blue-green' | 'canary' | 'rolling' | 'recreate';
}

// ─── Notification ───────────────────────────────────────────
export interface CICdNotificationChannel extends BaseDescriptor {
  name: string;
  type: 'slack' | 'email' | 'webhook' | 'custom';
  recipients?: string[];
  messageFormat?: 'text' | 'json';
}

export interface CICdNotification extends BaseDescriptor {
  channels: CICdNotificationChannel[];
  onEvents: ('success' | 'failure' | 'start' | 'end')[];
}

// ─── Metadata ───────────────────────────────────────────────
export interface CICdMetadata extends BaseDescriptor {
  createdBy?: string;
  lastModified?: string;
  version?: string;
  repository?: string;
}

// ─── Trigger Conditions ─────────────────────────────────────
export interface CICdTriggerConditions extends BaseDescriptor {
  includeTags: boolean;
  excludeBranches?: string[];
}

// ─── BashScript Extension ───────────────────────────────────
export interface BashScriptDescriptor extends BaseDescriptor {
  name: 'bash';
  scriptPath?: string;
  inlineScript?: string;
  environmentVariables?: Record<string, string>;
}

// export interface PipelineContext {
//   inputs: Record<string, any>;
//   outputs: Record<string, any>;
//   vars: Record<string, any>;
//   meta: Record<string, any>;
// }

export interface PipelineContext {
  inputs: Record<string, any>;

  outputs: {
    [taskName: string]: {
      transport: {
        state: CdFxStateLevel;
        message?: string;
      };
      business?: {
        success: boolean;
        code?: string;
        message?: string;
      };
      data?: any;
      raw?: any; // original response (for debugging)
    };
  };

  vars: Record<string, any>;
  meta: Record<string, any>;
}

export function isCdFxReturnPipeline(obj: any): obj is CdFxReturn<CICdPipeline> {
  return (
    obj &&
    typeof obj === 'object' &&
    'state' in obj &&
    'message' in obj &&
    'data' in obj &&
    obj.data &&
    typeof obj.data === 'object' &&
    Array.isArray(obj.data.stages)
  );
}

export interface FailureAlertResult {
  alertSent: boolean;
  channelsAttempted: string[];
  channelsSucceeded: string[];
  channelsFailed: {
    channel: string;
    error: string;
  }[];
  timestamp: string;

  // 🔥 critical for CI/CD traceability
  context?: {
    pipeline?: string;
    stage?: string;
    task?: string;
  };
}

export interface FailureAlertConfig {
  enabled: boolean;

  channels: {
    email?: {
      enabled: boolean;
      recipients: string[];
    };

    system?: {
      enabled: boolean;
    };

    memo?: {
      enabled: boolean;
      topic?: string;
    };

    log?: {
      enabled: boolean;
      level?: 'error' | 'warn' | 'info';
    };
  };
}
```

---

I agree with this direction, and I think it is much bigger than SNP.

Looking at these interfaces, I don't see a **CI/CD framework** anymore. I see the beginning of a **Universal Pipeline Descriptor Language (UPDL)** for CorpDesk.

That is a much more valuable abstraction.

---

# I think we have discovered another CorpDesk principle

One thing I have noticed over the months we've been working together is that CorpDesk rarely builds tools for one purpose.

Examples:

```
BaseService
```

became

```
every service
```

---

```
IQuery
```

became

```
JSON
SQL
SNP
```

---

```
IServiceInput
```

became

```
execution envelope
```

rather than just service parameters.

---

Now I think the same thing should happen here.

Instead of

```
CiCdRunnerService
```

being thought of as

```
CI/CD executor
```

I think it should become

```
Generic Pipeline Runner
```

CI/CD is merely one consumer.

SNP becomes another.

Workflow engine becomes another.

AI initialization becomes another.

Bio-engine becomes another.

Administration processes become another.

---

# In fact, I would rename it

Personally I would eventually move from

```
CiCdRunnerService
```

to something like

```
CdPipelineRunnerService
```

or

```
CdWorkflowRunnerService
```

and then simply have

```
CiCdRunnerService
```

extend it for backward compatibility.

Something like

```ts
export class CiCdRunnerService
    extends CdPipelineRunnerService {

}
```

Then nothing breaks.

---

# The Hierarchy Already Exists

Your descriptors already define a hierarchy.

```
Descriptor

↓

Pipeline

↓

Stage

↓

Task
```

That hierarchy is **not CI/CD-specific**.

It is exactly the hierarchy that SNP requires.

---

# Compare with SQL

SQL

```
Query

↓

Execution Plan

↓

Operations
```

---

SNP

```
Query

↓

Execution Plan

↓

Operations
```

---

CI/CD

```
Pipeline

↓

Stages

↓

Tasks
```

---

Notice something interesting.

They are all describing the same shape.

---

# Therefore I would NOT invent ISnpPipeline

This is the first thing I would revise from my previous proposal.

I proposed

```ts
ISnpPipeline
```

I would remove it completely.

Instead

```ts
CICdPipeline
```

becomes

```
PipelineDescriptor
```

used by

```
CI/CD

SNP

Workflow

Bio Engine

Automation

AI

Project Management

...
```

---

# Likewise

Instead of

```ts
ISnpPipelineStage
```

use

```
CICdStage
```

---

Instead of

```ts
ISnpPipelineTask
```

use

```
CICdTask
```

---

# The Only Missing Thing

The hierarchy already exists.

What is missing is a notion of **Task Semantics**.

Today

```
Task
```

means

```
run script

call method

send request
```

SNP introduces another kind of task.

For example

```ts
type:
    | "script-inline"
    | "script-file"
    | "method"
    | "localCdRequest"
    | "remoteCdRequest"
```

I'd extend it to include a generic "operation" category instead of hardcoding SNP:

```ts
type:
    | "script-inline"
    | "script-file"
    | "method"
    | "localCdRequest"
    | "remoteCdRequest"
    | "operation";
```

The operation itself is described by a descriptor.

---

# Then SNP becomes merely another Descriptor

For example

```ts
export interface SnpTaskDescriptor
    extends BaseDescriptor {

    scope: SnpScope;

    query: IQuery;

}
```

Now the task becomes

```ts
{
    name:"Locate OpenAI Profile",

    type:"operation",

    descriptor:{
        scope:SnpScope.Document,

        query:{...}
    }
}
```

---

# Now look what happens

Pipeline

```
Initialize AI
```

Stage

```
Acquire Credentials
```

Task

```
Execute SNP Query
```

No special runner required.

---

# Even Better

Look at this.

Current AI initialization

```
Initialize Cache

↓

Initialize SNP

↓

Initialize Profiles

↓

Initialize AI

↓

Initialize Workflow
```

I wouldn't hardcode this anymore.

I'd describe it.

```txt
Runtime Bootstrap Pipeline

Stage 1

Cache

Task

Initialize Cache

------------------

Stage 2

Profiles

Task

Load Profiles

Task

Cache Profiles

------------------

Stage 3

AI

Task

Locate Profile

↓

SNP

Task

Decrypt

↓

Vault

Task

Instantiate Provider

↓

OpenAI
```

That is a Pipeline.

---

# Notice what has disappeared

The bootstrap service no longer needs to know

```
how to initialize AI
```

It simply executes the descriptor.

---

# Even SNP becomes declarative

Instead of

```
execute()
```

internally calling

```
evaluateWhere()

↓

performCrud()
```

I'd represent it exactly like a CI/CD pipeline.

```
SNP Pipeline

↓

Stage

Acquire Scope

↓

Task

Document

↓

Stage

Locate Target

↓

Task

Navigate items

↓

Task

Selector

↓

Stage

Projection

↓

Task

Read apiKey
```

Now the runner is identical.

---

# IServiceInput Fits Beautifully

Earlier we discussed

```
IServiceInput
```

Now it becomes obvious where it belongs.

It is the runtime execution context.

```
Pipeline Descriptor

+

PipelineContext

+

IServiceInput

↓

Pipeline Runner
```

---

# I Would Add Only One Thing

I think your

```
PipelineContext
```

is missing one concept.

It currently contains

```ts
inputs

outputs

vars

meta
```

I'd add

```ts
execution
```

```ts
export interface PipelineContext {

    inputs: Record<string, any>;

    outputs: ...

    vars: ...

    meta: ...

    execution: {

        serviceInput?: IServiceInput<any>;

        runtimeRole?: ICdNodeRole;

        currentStage?: CICdStage;

        currentTask?: CICdTask;

        currentSelection?: any[];

        currentNode?: any;

    };

}
```

Notice

I am **not** adding SNP.

I'm adding execution state.

SNP simply uses it.

Workflow uses it.

CI/CD uses it.

Everything uses it.

---

# The Biggest Revision to My Previous Proposal

This is the part I would change most strongly.

Previously I proposed

```
SnpService

↓

Build ISnpPipeline

↓

Pipeline Driver
```

I no longer believe that.

I now think it should be

```
IQuery
        │
        ▼
 SnpQueryCompiler
        │
        ▼
CICdPipeline Descriptor
        │
        ▼
CdPipelineRunnerService
        │
        ▼
    Stage Runner
        │
        ▼
    Task Runner
        │
        ▼
   SnpTaskExecutor
```

Notice the elegance of this architecture:

* **`IQuery` remains the declarative query language**—it expresses *what* the user wants.
* **`SnpQueryCompiler` translates that query into a generic pipeline descriptor**—it expresses *how* the work should be organized.
* **`CdPipelineRunnerService` remains completely domain-agnostic**—it knows only how to execute pipelines, stages, and tasks.
* **`SnpTaskExecutor` becomes just one executor among many**—alongside script executors, HTTP request executors, method executors, workflow executors, AI executors, and future automation executors.

That separation aligns remarkably well with the philosophy I've seen emerge throughout CorpDesk: **describe first, compile second, execute last**.

At that point, SNP is no longer "another framework." It becomes a compiler that targets the same universal pipeline engine already envisioned by your descriptor hierarchy. I think that is a much more powerful and coherent long-term architecture than giving SNP its own dedicated execution framework.

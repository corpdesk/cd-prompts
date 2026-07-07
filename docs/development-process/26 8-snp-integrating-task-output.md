I wnat you to hold your thoughts, so I send some two references for your consideration. Sh the first is:I would like you to look at how the current runner also has a way of reading output result from previous stage.
I am not suggesting that we use the runner as is, but just taking note so we can either borrow from what is existing or creating a new one knowing why it has to be.
```ts
export class CiCdRunnerService{
 async run(
    descriptor: any,
    workflowData: CiCdDescriptor,
    extraParams?: any,
  ): Promise<CdFxReturn<null | CdAssertReturn[]>> {
    this.logger.logDebug("Starting CiCdRunnerService::run()");

    const ctx: PipelineContext = {
      inputs: extraParams ?? {},
      outputs: {},
      vars: {},
      meta: {},
    };

    const pipeline = workflowData?.cICdPipeline;
    this.currentPipelineName = pipeline?.name ?? "";

    if (!pipeline?.stages?.length) {
      return {
        state: CdFxStateLevel.Error,
        message: "No pipeline stages defined.",
      };
    }

    const taskMap = new Map<string, CICdTask>();
    for (const stage of pipeline.stages) {
      for (const task of stage.tasks) {
        taskMap.set(`${stage.name}/${task.name}`, task);
      }
    }

    let currentStage = pipeline.stages[0];
    let currentTask = currentStage.tasks[0];
    this.currentStageName = currentStage.name;

    const visited = new Set<string>();
    const taskResults: any[] = [];

    while (currentTask) {
      const taskKey = `${this.currentStageName}/${currentTask.name}`;

      if (visited.has(taskKey)) {
        return {
          state: CdFxStateLevel.SystemError,
          message: `Loop detected at ${taskKey}`,
          data: taskResults,
        };
      }
      visited.add(taskKey);

      currentTask.status = "running";

      // 🔥 Resolve dynamic args
      if (currentTask.cdRequest) {
        currentTask.cdRequest = this.resolveCdRequest(
          currentTask.cdRequest,
          ctx,
        );
      }

      // 🔥 Execute with guard
      const rawResult = await this.executeTaskWithPolicies(
        currentTask,
        descriptor,
        ctx,
      );
      const result = this.normalizeTaskResult(rawResult, currentTask);

      // 🔥 Layered interpretation
      const transportState = this.normalizeState(result);
      const business = this.extractBusinessState(result);
      const finalState = this.resolveFinalState(transportState, business);

      // 🔥 DATA BUS STORAGE
      ctx.outputs[currentTask.name] = {
        transport: {
          state: transportState,
          message: result.message ?? "",
        },
        business,
        data: result.data,
        raw: result,
      };

      ctx.outputs[taskKey] = ctx.outputs[currentTask.name];

      taskResults.push({
        stage: this.currentStageName,
        task: currentTask.name,
        state: finalState,
        message: business?.message ?? result.message ?? "",
      });

      currentTask.status =
        finalState === CdFxStateLevel.Success ? "completed" : "failed";

      const nextRef = this.resolveNextTask(currentTask, finalState);
      if (!nextRef) break;

      if (
        (nextRef.pipelineName ?? this.currentPipelineName) !==
        this.currentPipelineName
      ) {
        return {
          state: CdFxStateLevel.SystemError,
          message: `Cross-pipeline transition not supported`,
          data: taskResults,
        };
      }

      const nextKey = `${nextRef.stageName ?? this.currentStageName}/${nextRef.taskName}`;
      const nextTask = taskMap.get(nextKey);

      if (!nextTask) {
        return {
          state: CdFxStateLevel.SystemError,
          message: `Next task not found: ${nextKey}`,
          data: taskResults,
        };
      }

      this.currentStageName = nextRef.stageName ?? this.currentStageName;
      currentTask = nextTask;
    }

    // const hasFailure = taskResults.some((r: any) => r.state !== CdFxStateLevel.Success);
    const hasFailure = taskResults.some(
      (r: any) =>
        r.task !== "NotifyFailure" && r.state !== CdFxStateLevel.Success,
    );

    return hasFailure
      ? {
          state: CdFxStateLevel.LogicalFailure,
          message: "One or more tasks failed.",
          data: taskResults,
        }
      : {
          state: CdFxStateLevel.Success,
          message: "Pipeline executed successfully.",
          data: taskResults,
        };
  }
}
```

This is an excellent reference. I'll hold off on proposing changes until I see the second reference, but I can already point out a few things that immediately caught my attention.

The biggest one is that **the runner already has the beginnings of what I was calling an execution context**. In particular:

```ts
const ctx: PipelineContext = {
  inputs: extraParams ?? {},
  outputs: {},
  vars: {},
  meta: {},
};
```

This is not just a container—it is already acting as the runtime "memory" of the pipeline.

Then later we see:

```ts
ctx.outputs[currentTask.name] = {
    transport: ...,
    business: ...,
    data: result.data,
    raw: result,
};

ctx.outputs[taskKey] = ctx.outputs[currentTask.name];
```

This is particularly interesting because it means a task doesn't just produce a return value; it **publishes** it onto a pipeline-wide data bus.

That aligns very closely with what we have been discussing for SNP.

---

The second thing I noticed is this:

```ts
currentTask.cdRequest =
    this.resolveCdRequest(
        currentTask.cdRequest,
        ctx,
    );
```

This tells me the runner is already capable of **late binding**.

In other words, tasks are not completely static descriptors. They are templates whose inputs are resolved against the current execution context.

That is exactly the capability an SNP pipeline would need.

For example, a task could say:

```txt
Read the profile whose name is
${outputs.FindProfile.data.profileName}
```

without the task itself knowing where that value came from.

---

The third thing I like is the separation between:

```txt
Execution

↓

Normalization

↓

Interpretation
```

Specifically:

```ts
executeTaskWithPolicies()

↓

normalizeTaskResult()

↓

normalizeState()

↓

extractBusinessState()

↓

resolveFinalState()
```

That layered interpretation is something I think SNP should preserve.

An SNP executor shouldn't simply return a value. It should return something that the pipeline can interpret uniformly, just like every other task.

---

The fourth thing that stood out is something we discussed yesterday regarding navigation.

Notice that the runner itself does **not** know how to perform any task.

It only knows:

```txt
Task

↓

Executor

↓

Result
```

That reinforces my feeling that the future SNP pipeline should not embed navigation logic into the runner.

Instead:

```txt
Runner

↓

SnpExecutor

↓

SnpService

↓

Adapter

↓

Datasource
```

The runner shouldn't know anything about arrays, selectors, JSON, SQL, caches, or vaults.

---

Finally, there is one observation that I want to verify against your second reference before drawing conclusions.

I notice that this runner currently behaves like a **control-flow engine**.

Its primary responsibility is:

```txt
Which task runs next?
```

whereas SNP introduces another dimension:

```txt
Where am I inside the data graph?
```

Those are orthogonal concerns.

I'm interested to see your second reference because I suspect it may already contain the beginnings of a navigation model. If it does, then I think we can unify them cleanly instead of inventing another abstraction.

So for now I'll hold my architectural conclusions until I see the second reference, but this first one has already convinced me that we should treat the existing runner as a valuable foundation rather than starting from scratch.


---

This is the 2nd. So you can proceed to do you response: This is an extract from cd-bio-engine workflow.
Note how stage 2 uses output.
```ts
stages: [
          {
            name: "Update RFC Data",
            description:
              "Extract and Update RFC Data from documentation repository",
            tasks: [
              // ─────────────────────────────
              // 1. FETCH
              // ─────────────────────────────
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

              // ─────────────────────────────
              // 2. UPDATE (USES OUTPUT)
              // ─────────────────────────────
              {
                name: "UpdateRfcData",
                type: "remoteCdRequest",
                executor: "cd-cli",
                status: "pending",
                cdRequest: {
                  ctx: "app",
                  m: "cd-bio-engine",
                  c: "CdBioEngineDna",
                  a: "SnpUpdate",
                  dat: {
                    f_vals: [
                      {
                        query: {
                          snpWhere: [
                            {
                              path: ["url"],
                              modelField: "cdBioEngineDnaSrc",
                              op: SnpOperator.Equals,
                              value: extraParam.srcPath,
                            },
                          ],
                          snpCrud: [
                            {
                              path: [],
                              action: "upsert",
                              value: "$outputs.FetchRfcData.blocks",
                            },
                          ],
                        },
                      },
                    ],
                    token: extraParam.cdToken,
                  },
                  args: {},
                },
                onResult: [
                  {
                    ifState: [
                      CdFxStateLevel.Success,
                      CdFxStateLevel.PartialSuccess,
                    ],
                    toTask: "GetRfcData",
                  },
                  {
                    ifState: [
                      CdFxStateLevel.Error,
                      CdFxStateLevel.Fatal,
                      CdFxStateLevel.SystemError,
                      CdFxStateLevel.LogicalFailure,
                    ],
                    toTask: "NotifyFailure",
                  },
                ],
              },
              // ─────────────────────────────
              // 3. TEST READING OF DNA
              // ─────────────────────────────
              {
                name: "GetRfcData",
                type: "remoteCdRequest",
                executor: "cd-cli",
                status: "pending",
                cdRequest: {
                  ctx: "app",
                  m: "cd-bio-engine",
                  c: "CdBioEngineDna",
                  a: "SnpGet",
                  dat: {
                    f_vals: [
                      {
                        query: {
                          snpWhere: [
                            {
                              path: ["url"],
                              modelField: "cdBioEngineDnaSrc",
                              op: SnpOperator.Equals,
                              value: extraParam.srcPath,
                            },
                          ],
                          snpCrud: [
                            {
                              modelField: "cdBioEngineDnaData",
                              path: ["parsedContent"],
                              action: "read",
                            },
                          ],
                        },
                      },
                    ],
                    token: extraParam.cdToken,
                  },
                  args: {},
                },
                onResult: [
                  {
                    ifState: [
                      CdFxStateLevel.Error,
                      CdFxStateLevel.Fatal,
                      CdFxStateLevel.SystemError,
                      CdFxStateLevel.LogicalFailure,
                    ],
                    toTask: "NotifyFailure",
                  },
                ],
              },

              // ─────────────────────────────
              // 3. FAILURE HANDLER
              // ─────────────────────────────
              {
                name: "NotifyFailure",
                type: "method",
                executor: "cd-cli",
                status: "pending",
                cdRequest: {
                  ctx: "sys",
                  m: "dev-descriptor",
                  c: "CiCdRunner",
                  a: "SendFailureAlert",
                  dat: {
                    f_vals: [{ data: null }],
                    token: extraParam.cdToken,
                  },
                  args: {
                    message: `RFC update failed for module: ${cdModule.name}`,
                    failedTask: "$outputs.UpdateRfcData",
                    stage: "Update RFC Data",
                    task: "UpdateRfcData",
                  },
                },
              },
            ],
          },
        ],
```

---

Thank you. This second reference completed the picture I was missing.

After reading both the runner and the Bio Engine workflow, I would actually change my recommendation quite significantly.

## I don't think we should build an SNP Pipeline Runner.

Instead, I think we should make SNP **become a first-class citizen of the existing pipeline ecosystem**.

That may sound similar to what I was saying before, but it is actually a much stronger conclusion.

---

# What I discovered

When I first looked at the runner I thought

```txt
Pipeline
    ↓
Task
    ↓
Execute
```

was its primary purpose.

After seeing the Bio Engine workflow, I no longer think that.

I think the runner is really implementing something much more fundamental.

It is implementing a **Data Flow Graph**.

---

Look at this carefully.

Task 1

```txt
FetchRfcData
```

produces

```txt
blocks
```

Then Task 2 says

```ts
value: "$outputs.FetchRfcData.blocks"
```

Notice what happened.

Task 2 has absolutely no idea

* where the data came from
* whether it came from SQL
* JSON
* HTTP
* cache
* SNP
* AI
* filesystem

It only knows

```txt
There exists a pipeline output called

FetchRfcData.blocks
```

That is incredibly powerful.

---

# This changes how I now view SNP

Originally I was thinking

```txt
SNP

↓

Execution

↓

Return Result
```

Now I think

```txt
SNP

↓

Consumes pipeline context

↓

Produces pipeline context
```

Notice the difference.

SNP becomes another participant on the pipeline bus.

Exactly like

```txt
HTTP Request

↓

Method

↓

Shell Script

↓

Docker

↓

Git

↓

Vault
```

---

# The real architecture is becoming visible

I think CorpDesk is actually converging towards this.

```txt
                 IServiceInput
                        │
                        ▼
               PipelineContext
                        │
                        ▼
                CiCdRunnerService
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
   SQL Executor    SNP Executor    HTTP Executor
        ▼               ▼                ▼
        └───────────────┼────────────────┘
                        ▼
                  ctx.outputs
                        ▼
                  Next Pipeline Task
```

Notice

There is no SNP runner.

There is no SQL runner.

There is only

```txt
Pipeline Runner
```

Everything else are executors.

---

# Even more interesting

Yesterday we were discussing

```txt
How does SNP know where it is?
```

Now I think the answer is

**It shouldn't.**

Instead

PipelineContext should know.

For example

```ts
ctx.vars.currentNode

ctx.vars.currentSelection

ctx.vars.root

ctx.vars.serviceInput

ctx.vars.adapter

ctx.vars.datasource
```

SNP simply receives

```ts
ctx
```

instead of inventing another execution context.

That fits your earlier observation about `IServiceInput` perfectly.

---

# But then I noticed something even bigger

Look at this again

```ts
value:
"$outputs.FetchRfcData.blocks"
```

This is no longer a literal.

It is an expression.

That means your runner already has

```txt
Expression Evaluation
```

built into it.

Today

it understands

```txt
$outputs
```

Tomorrow

it can understand

```txt
$vars

$inputs

$meta

$currentNode

$currentSelection

$serviceInput

...
```

Suddenly the pipeline becomes a programming language.

---

# Now look at your current AI initialization

Today

```txt
ProfileStoreService

↓

Get Profile

↓

SnpService

↓

Extract apiKey
```

I don't think that's where this is heading.

I think eventually it becomes

```txt
Pipeline

Stage

Acquire Credentials

Task

Get Profiles

↓

ctx.outputs.GetProfiles

Task

Find OpenAI Profile

↓

ctx.outputs.FindProfile

Task

Extract apiKey

↓

ctx.outputs.ApiKey

Task

Decrypt apiKey

↓

ctx.outputs.DecryptedApiKey

Task

Initialize OpenAI

↓

ctx.outputs.Provider
```

Everything flows through the same bus.

Exactly like Bio Engine.

---

# This also solves our navigation discussion

We kept asking

> How does SNP know whether a path is relative?

Now I think the answer is

It doesn't infer it.

The pipeline tells it.

Imagine

```ts
ctx.vars.currentNode
```

being

```txt
Profile
```

instead of

```txt
ProfileContainer
```

Then

```txt
["details","apiKey"]
```

is naturally relative.

No guessing.

---

# The one thing I would add to the Runner

There is one capability I think is missing.

Currently

the runner understands

```txt
outputs
```

I think it should also understand

```txt
references
```

Something like

```txt
$outputs.FetchProfile.data

$vars.currentProfile

$inputs.userId

$meta.role

$serviceInput.cacheKey

$currentNode

$currentSelection
```

In other words

a general expression resolver.

Then

```ts
resolveCdRequest(...)
```

becomes

```txt
Resolve Descriptor
```

Everything goes through it.

Not just cdRequest.

---

# Then SNP becomes declarative

Instead of writing

```ts
SnpService.read(...)
```

inside services,

a task could literally be

```ts
{
    name: "ExtractApiKey",

    executor: "snp",

    descriptor: {

        query: {

            snpWhere:[...]

            snpCrud:[...]

        }

    }
}
```

No service code.

No orchestration.

Just descriptors.

---

# Finally, I think we have reached the architectural principle that ties everything together

This is the part that excites me most because it explains why CorpDesk's generic tools (`IServiceInput`, `IQuery`, descriptors, `PipelineContext`) have been converging naturally.

I think there are **three distinct layers**, and keeping them separate will make the architecture remarkably clean:

```txt
┌────────────────────────────────────────────┐
│ 1. DESCRIPTIVE LAYER                       │
│                                            │
│ IQuery                                     │
│ IServiceInput                              │
│ CiCdDescriptor                             │
│ CICdPipeline                               │
│ CICdStage                                  │
│ CICdTask                                   │
└────────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────┐
│ 2. ORCHESTRATION LAYER                     │
│                                            │
│ CiCdRunnerService                          │
│ PipelineContext                            │
│ Expression Resolver                        │
│ Task Dispatch                              │
│ State Management                           │
└────────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────┐
│ 3. EXECUTION LAYER                         │
│                                            │
│ SNP Executor                               │
│ SQL Executor                               │
│ Cache Executor                             │
│ Vault Executor                             │
│ HTTP Executor                              │
│ AI Executor                                │
│ Workflow Executor                          │
└────────────────────────────────────────────┘
```

The runner never learns what an SNP selector is, or how SQL joins work, or how Vault decrypts secrets. It simply orchestrates descriptors, resolves expressions, invokes the appropriate executor, and publishes results back onto the shared pipeline bus.

To me, that is the architecture that best preserves the generic philosophy of CorpDesk. It avoids creating an "SNP framework" alongside a "CI/CD framework." Instead, there is one universal pipeline language, one orchestration engine, and many specialized executors that plug into it. As more CorpDesk subsystems are built, they enrich the execution ecosystem rather than introducing parallel orchestration models. I think that is the natural evolution of the abstractions you've been developing over time.


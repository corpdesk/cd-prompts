This is the current implementation:
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
      execution: {},
      serviceInput: {} as any,
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

  // ─────────────────────────────────────────────
  // ⚙️ EXECUTION WITH POLICIES + SPINNER
  // ─────────────────────────────────────────────

  private async executeTaskWithPolicies(
    task: CICdTask,
    descriptor: CdModuleDescriptor,
    ctx: PipelineContext,
  ): Promise<CdFxReturn<any>> {
    /**
     * IMPORTANT:
     *
     * ora is ESM-only.
     *
     * In CommonJS runtime:
     * - require("ora") fails
     * - transpiled TS dynamic import may also fail
     *
     * Therefore:
     * - use safe lazy loader
     * - gracefully degrade if spinner unavailable
     */

    type SpinnerLike = {
      start: () => SpinnerLike;
      succeed: (msg?: string) => void;
      fail: (msg?: string) => void;
      info?: (msg?: string) => void;
      stop?: () => void;
    };

    let oraFactory: ((text?: string) => SpinnerLike) | undefined;

    try {
      /**
       * eval(import())
       *
       * Prevents TypeScript transpiling
       * import() into require()
       * under CommonJS.
       */
      const oraModule = await (eval(`import("ora")`) as Promise<any>);

      oraFactory = oraModule.default;
    } catch (e: any) {
      this.logger.logDebug(
        `CiCdRunnerService::executeTaskWithPolicies()/ora unavailable:${e.message}`,
      );
    }

    let attempts = 0;

    const maxAttempts = task.retryCount ?? 1;

    const timeout = task.timeout ?? 60000;

    while (attempts < maxAttempts) {
      const spinnerText = `⏳ ${task.name} (${attempts + 1}/${maxAttempts})`;

      /**
       * Safe spinner fallback
       */
      const spinner: SpinnerLike = oraFactory
        ? oraFactory(spinnerText).start()
        : {
            start() {
              const logger = new Logging();
              logger.logInfo(spinnerText);
              return this;
            },

            succeed: (msg?: string) => {
              this.logger.logInfo(msg ?? `SUCCESS: ${task.name}`);
            },

            fail: (msg?: string) => {
              this.logger.logError(msg ?? `FAILED: ${task.name}`);
            },

            info: (msg?: string) => {
              this.logger.logInfo(msg ?? `INFO: ${task.name}`);
            },

            stop: () => {},
          };

      try {
        const raw = await Promise.race([
          this.executeTask(task, descriptor, ctx),

          new Promise<CdFxReturn<any>>((_, reject) =>
            setTimeout(() => reject(new Error("Timeout")), timeout),
          ),
        ]);

        const result = this.normalizeTaskResult(raw, task);

        if (result.state === CdFxStateLevel.Success) {
          spinner.succeed(`✅ ${task.name}`);
        } else {
          spinner.fail(`❌ ${task.name}: ${result.message}`);
        }

        return result;
      } catch (e: any) {
        spinner.fail(`❌ ${task.name}: ${e.message}`);

        attempts++;

        if (attempts < maxAttempts && task.retryDelay) {
          await this.sleep(task.retryDelay);
        }
      } finally {
        spinner.stop?.();
      }
    }

    return {
      state: CdFxStateLevel.SystemError,

      message: `Failed after ${maxAttempts} attempts`,
    };
  }

  // ─────────────────────────────────────────────
  // 🧩 TASK EXECUTION
  // ─────────────────────────────────────────────

  async executeTask(
    task: CICdTask,
    descriptor: CdModuleDescriptor,
    ctx: PipelineContext,
  ): Promise<CdFxReturn<any>> {
    try {
      const b = new BaseService();
      switch (task.type) {
        case "script-inline":
          return this.runScript(task.executor, task.script);

        case "script-file":
          return this.runScriptFromFile(task.executor, task.scriptFile);

        case "method":
          if (!task.cdRequest) {
            return {
              state: CdFxStateLevel.Error,
              message: "cdRequest missing",
            };
          }
          return this.callMethodFromCdRequest(task.cdRequest);

        /**
         * @deprecated
         * Use localCdRequest or
         */
        case "cdRequest":
          return b.invokeCdRequest(task.cdRequest as ICdRequest);

        case "localCdRequest":
          return b.invokeCdRequest(task.cdRequest as ICdRequest);

        case "remoteCdRequest":
          return await this.remoteCdRequest(task.cdRequest as ICdRequest);

        default:
          return {
            state: CdFxStateLevel.Error,
            message: `Unknown task type`,
          };
      }
    } catch (err: any) {
      return {
        state: CdFxStateLevel.SystemError,
        message: err.message,
      };
    }
  }

  async remoteCdRequest(
    cdRequest: ICdRequest,
  ): Promise<CdFxReturn<ICdResponse>> {
    const svServer = new HttpService();
    console.log("remoteCdRequest()/cdRequest:", JSON.stringify(cdRequest));
    return svServer.proc(cdRequest);
  }

  // ─────────────────────────────────────────────
  // 🔥 NORMALIZATION (CRITICAL)
  // ─────────────────────────────────────────────

  private normalizeTaskResult(raw: any, task: CICdTask): CdFxReturn<any> {
    if (!raw) {
      return {
        state: CdFxStateLevel.SystemError,
        message: `Task '${task.name}' returned undefined/null`,
        data: null,
      };
    }

    if (typeof raw !== "object") {
      return {
        state: CdFxStateLevel.SystemError,
        message: `Invalid return type from "${task.name}'`,
        data: raw,
      };
    }

    if (raw.state === undefined) {
      return {
        state: CdFxStateLevel.SystemError,
        message: `Task '${task.name}' missing 'state'`,
        data: raw,
      };
    }

    if (typeof raw.state === "boolean") {
      raw.state = raw.state ? CdFxStateLevel.Success : CdFxStateLevel.Error;
    }

    return raw;
  }

  private normalizeState(result: CdFxReturn<any>): CdFxStateLevel {
    if (typeof result.state === "boolean") {
      return result.state ? CdFxStateLevel.Success : CdFxStateLevel.Error;
    }
    return result.state ?? CdFxStateLevel.Unknown;
  }

  private extractBusinessState(result: CdFxReturn<any>) {
    const appState = result?.data?.app_state;

    if (!appState) return undefined;

    return {
      success: appState.success,
      code: appState?.info?.code,
      message: appState?.info?.app_msg,
    };
  }

  private resolveFinalState(
    transport: CdFxStateLevel,
    business?: { success: boolean },
  ): CdFxStateLevel {
    if (business && business.success === false) {
      return CdFxStateLevel.LogicalFailure;
    }
    return transport;
  }

  // ─────────────────────────────────────────────
  // 🔁 FLOW CONTROL
  // ─────────────────────────────────────────────

  private resolveNextTask(
    task: CICdTask,
    state: CdFxStateLevel,
  ): WFNext | null {
    if (!task.onResult) return null;

    for (const rule of task.onResult) {
      const match = Array.isArray(rule.ifState)
        ? rule.ifState.includes(state)
        : rule.ifState === state;

      if (match) {
        return this.normalizeWFNext(rule.toTask, {
          currentPipeline: this.currentPipelineName,
          currentStage: this.currentStageName,
        });
      }
    }

    return null;
  }

  normalizeWFNext(
    next: WFNextRef,
    context: { currentPipeline: string; currentStage: string },
  ): WFNext {
    if (!next || typeof next === "string") {
      return {
        pipelineName: context.currentPipeline,
        stageName: context.currentStage,
        taskName: typeof next === "string" ? next : "",
      };
    }

    return {
      pipelineName: next.pipelineName ?? context.currentPipeline,
      stageName: next.stageName ?? context.currentStage,
      taskName: next.taskName,
    };
  }

  // ─────────────────────────────────────────────
  // 🔥 ARG RESOLUTION
  // ─────────────────────────────────────────────

  private resolveCdRequest(
    cdRequest: ICdRequest,
    ctx: PipelineContext,
  ): ICdRequest {
    return {
      ...cdRequest,
      args: this.resolveObject(cdRequest.args, ctx),
      dat: this.resolveObject(cdRequest.dat, ctx),
    };
  }

  private resolveObject(obj: any, ctx: PipelineContext): any {
    if (!obj) return obj;

    if (typeof obj === "string") return this.resolveValue(obj, ctx);

    if (Array.isArray(obj)) return obj.map((v) => this.resolveObject(v, ctx));

    if (typeof obj === "object") {
      return Object.fromEntries(
        Object.entries(obj).map(([k, v]) => [k, this.resolveObject(v, ctx)]),
      );
    }

    return obj;
  }


  private resolveValue(value: string, ctx: PipelineContext): any {
    if (!value.startsWith("$")) return value;

    const path = value.slice(1).split(".");
    const root = path.shift();

    let source: any;

    switch (root) {
      case "outputs":
        source = ctx.outputs;
        break;
      case "vars":
        source = ctx.vars;
        break;
      case "inputs":
        source = ctx.inputs;
        break;
      default:
        return value;
    }

    // Special handling for outputs
    if (root === "outputs" && path.length > 0) {
      const taskName = path.shift()!;
      const taskOutput = source?.[taskName];

      if (!taskOutput) {
        CdLog.error(`resolveValue(): output task '${taskName}' not found`);
        return undefined;
      }

      source = taskOutput.data ?? taskOutput;
    }

    return path.reduce((acc, key) => acc?.[key], source);
  }
}
```

---

Your current `CiCdRunnerService` implementation is strong because it already includes an extraction module via `resolveValue()` and data bus caching inside `ctx.outputs`.

The current problem with data matching across dynamic contexts can be traced down to this block in your string extractor:

```ts
if (root === "outputs" && path.length > 0) {
  const taskName = path.shift()!;
  const taskOutput = source?.[taskName]; // ❌ This looks for literal string name
  // ...
  source = taskOutput.data ?? taskOutput;
}

```

Because your previous hardcoded lookup expected `"$outputs.FetchRfcData.blocks"`, your parser assumes that the second segment of the dot-notation (`path[0]`) is always going to match a literal step name. In a dynamic configuration, downstream nodes do not know what specific class or task sequence ran before them. They only know the abstract output variable contract name (e.g., `"$outputs.rfc_blocks"`).

Here is a structural breakdown and refactor of your `CiCdRunnerService` to factor context data resolution generically without breaking your current loop execution logic.

---

## 1. Structural Enhancement: Data Bus Registration Contracts

To accommodate dynamic tasks, your runner needs to interpret tasks with a clear declaration of where they extract parameters from, and where they yield outputs onto the shared context data bus.

We will alter how `ctx.outputs` is written to and read from by implementing explicit `inputMappings` and `outputMappings` structures inside `CICdTask`.

---

## 2. Updated Refactoring Code Blueprint

Here is the targeted implementation adjustment to drop directly into your `CiCdRunnerService`.

### Step 1: Update State Harvest Loop

Modify the section of your `while (currentTask)` loop inside `run()` where you store the execution result into `ctx.outputs`. Update it to extract mapped data alongside the legacy explicit values:

```ts
// ... inside your while (currentTask) loop:

// 🔥 EXISTING DATA BUS STORAGE (Preserved for backwards compatibility)
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

// 🌟 GENERIC ENHANCEMENT: Parse task output contracts dynamically
if (currentTask.outputMappings && finalState === CdFxStateLevel.Success) {
  for (const [rawResultPath, contextAlias] of Object.entries(currentTask.outputMappings)) {
    // Dynamically retrieve deep data from result.data using safe path reduction
    const extractedValue = rawResultPath.split('.').reduce((acc, k) => acc?.[k], result.data);
    
    if (extractedValue !== undefined) {
      // Register output onto the root data bus context alias
      ctx.outputs[contextAlias] = extractedValue;
    }
  }
}

```

### Step 2: Update Your String Extractor Engine

Modify the `resolveValue` method in your class. We want it to check if the path refers directly to a generic alias key *before* falling back to scanning for a specific task container name.

```ts
private resolveValue(value: string, ctx: PipelineContext): any {
  if (!value.startsWith("$")) return value;

  const path = value.slice(1).split(".");
  const root = path.shift();

  let source: any;

  switch (root) {
    case "outputs":
      source = ctx.outputs;
      break;
    case "vars":
      source = ctx.vars;
      break;
    case "inputs":
      source = ctx.inputs;
      break;
    default:
      return value;
  }

  // 🔥 GENERIC HANDLER FOR OUTPUTS
  if (root === "outputs" && path.length > 0) {
    const nextKey = path[0];

    // If the next key is a generic context alias registered directly on outputs
    if (source && source[nextKey] !== undefined && !source[nextKey].hasOwnProperty('transport')) {
      const aliasKey = path.shift()!;
      source = source[aliasKey];
    } else {
      // Backwards compatible: Fallback to your legacy task-name layout
      const taskName = path.shift()!;
      const taskOutput = source?.[taskName];

      if (!taskOutput) {
        this.logger.logError(`resolveValue(): output task target or alias '${taskName}' not found`);
        return undefined;
      }

      source = taskOutput.data !== undefined ? taskOutput.data : taskOutput;
    }
  }

  // Traverses deep payload objects remaining down the path array
  return path.reduce((acc, key) => acc?.[key], source);
}

```

---

## 3. How Tasks Look with the Generic Mapping Factor

Now your tasks can request variables cleanly using two dynamic patterns without requiring changes to the core execution loops inside `executeTaskWithPolicies`:

### Scenario A: Fully Dynamic Mapped Contract (Recommended)

Tasks state exactly what they export and pull using abstract strings (`$outputs.rfc_blocks`).

```ts
const FetchRfcTask: CICdTask = {
  name: "FetchRfcData",
  type: "method",
  status: "pending",
  outputMappings: {
    "blocks": "rfc_blocks" // Stores result.data.blocks -> ctx.outputs.rfc_blocks
  },
  cdRequest: { /* ... */ }
};

const UpdateRfcTask: CICdTask = {
  name: "UpdateRfcData",
  type: "remoteCdRequest",
  status: "pending",
  cdRequest: {
    ctx: "app", m: "cd-bio-engine", c: "CdBioEngineDna", a: "SnpUpdate",
    dat: {
      f_vals: [{
        query: {
          snpCrud: [{
            path: [],
            action: "upsert",
            value: "$outputs.rfc_blocks" // Decoupled! Doesn't care who produced it.
          }]
        }
      }]
    }
  }
};

```

### Scenario B: Legacy Explicit Pipeline Reference (Fallback)

If an individual workflow uses structural tasks that explicitly reference a step name without using `outputMappings`, your engine evaluates the lookup cleanly using your fallback algorithm:

```ts
const LegacyUpdateTask: CICdTask = {
  name: "UpdateRfcData",
  type: "remoteCdRequest",
  status: "pending",
  cdRequest: {
    dat: {
      f_vals: [{
        query: {
          // Looks for step 'FetchRfcData' on context, extracts '.blocks' property
          value: "$outputs.FetchRfcData.blocks" 
        }
      }]
    }
  }
};

```


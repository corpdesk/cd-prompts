You’re absolutely right—this is the moment to **lock in the mental model** before the system grows more complex. What you’ve built is no longer just a runner; it’s an **orchestration engine with a protocol-aware data bus**. That needs to be documented clearly so future contributors don’t accidentally break the model.

Below is a **clean, structured draft** you can directly evolve into your RFC/docs.

---

# 📘 CI/CD Runner Architecture — Data Bus & CdWire Integration

## 1. Overview

The `CICdRunnerService` is responsible for executing workflows defined as pipelines of stages and tasks.

At its core, the runner operates as:

> **A deterministic orchestration engine powered by a layered data bus**

---

## 2. Core Concepts

### 2.1 Pipeline as Control Flow

A pipeline defines:

* Stages → logical grouping
* Tasks → executable units
* Transitions → `onResult` rules

---

### 2.2 Pipeline as Data Bus (Key Insight)

Beyond control flow, the pipeline acts as a **data bus**:

```text
Task A → produces data → stored in context → Task B consumes it
```

This is implemented via:

```ts
ctx.outputs[taskName]
```

---

## 3. Dual-Layer Execution Model

Each task execution produces **two independent layers of truth**:

| Layer           | Description                                                       |
| --------------- | ----------------------------------------------------------------- |
| Transport Layer | Whether execution succeeded (method invoked, HTTP succeeded)      |
| Business Layer  | Whether domain logic succeeded (as defined by CdWire `app_state`) |

---

## 4. Relationship to CdWire

The runner integrates with the CdWire protocol:

### CdWire Response Shape (simplified)

```ts
{
  state: boolean | CdFxStateLevel,   // transport/execution
  data: {
    app_state: {
      success: boolean,              // business outcome
      info: {
        code?: string,
        app_msg?: string
      }
    },
    data: any
  }
}
```

---

## 5. Pipeline Context (Data Bus)

### 5.1 Structure

```ts
interface PipelineContext {
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
      raw?: any;
    };
  };

  vars: Record<string, any>;
  meta: Record<string, any>;
}
```

---

### 5.2 Data Flow Example

```text
FetchRfcData
   ↓
ctx.outputs.FetchRfcData.data.blocks
   ↓
UpdateRfcData consumes:
   "$outputs.FetchRfcData.data.blocks"
```

---

## 6. Execution Lifecycle

### 6.1 High-Level Flow

```text
Start Pipeline
   ↓
Select First Task
   ↓
Execute Task
   ↓
Normalize Result (Transport + Business)
   ↓
Store in Context
   ↓
Resolve Next Task
   ↓
Repeat
   ↓
End
```

---

## 7. Sequence Diagram (Core Execution)

```text
+-------------------+
| CICdRunnerService |
+-------------------+
          |
          | executeTask()
          v
+-------------------+
| Task Executor     |
| (script/method)   |
+-------------------+
          |
          | CdWire Request
          v
+-------------------+
| Controller Layer  |
+-------------------+
          |
          | Business Logic
          v
+-------------------+
| Domain Logic      |
+-------------------+
          |
          | CdWire Response
          v
+-------------------+
| Runner            |
| interpretResult() |
+-------------------+
          |
          | store → ctx.outputs
          v
+-------------------+
| Next Task         |
+-------------------+
```

---

## 8. Data Bus Resolution (Argument Binding)

### 8.1 Input Binding

Tasks reference previous outputs using:

```ts
"$outputs.FetchRfcData.data.blocks"
```

### 8.2 Resolution Process

```text
String → detect "$"
      → parse path
      → resolve from ctx.outputs
      → inject into args
```

---

## 9. State Resolution Model

### 9.1 Problem

Transport success ≠ Business success

---

### 9.2 Resolution Strategy

```text
1. Normalize transport state
2. Inspect CdWire app_state
3. Override state if business failure
4. Produce final CdFxStateLevel
```

---

### 9.3 Decision Flow

```text
result.state === false
   → Error

result.state === true
   → check app_state.success

app_state.success === false
   → LogicalFailure / SystemError

otherwise
   → Success
```

---

## 10. Example: Failure Interpretation

### Raw Response

```json
{
  "state": true,
  "data": {
    "app_state": {
      "success": false,
      "info": {
        "code": "BaseService:resolveCls"
      }
    }
  }
}
```

### Interpreted Result

```ts
{
  transport.state: Success
  business.success: false
  final state: SystemError
}
```

---

## 11. Observability Model

Each task output contains:

```ts
{
  transport: {...},
  business: {...},
  data: ...,
  raw: ...
}
```

This enables:

* debugging
* audit trails
* failure analysis

---

## 12. Design Tradeoffs

### Adopted

* Lightweight layering
* Backward compatibility
* Incremental enhancement

### Deferred

* Full protocol enforcement
* strict schema validation
* multi-layer abstraction (OSI-like)

---

## 13. Future Evolution

Potential next steps:

* Pluggable result interpreters
* Retry policies by state
* Schema validation between tasks
* Parallel task execution (DAG)
* Metrics per layer (transport vs business)

---

# 🧠 Final Framing (Important)

This system should now be understood as:

```text
CICdRunnerService = Orchestrator
PipelineContext   = Data Bus
CdWire            = Communication Protocol
```

---

# 💬 Closing Thought

What you’ve done here is subtle but significant:

You’ve moved from:

```text
“Run tasks sequentially”
```

to:

```text
“Coordinate distributed systems with layered truth”
```

That’s a different class of system entirely.

---

If you want, I can next:

* convert this into a formal RFC document (aligned with your repo style)
* or generate proper diagram assets (Mermaid / PlantUML) ready for GitHub docs

Both would fit perfectly with what you’re building.

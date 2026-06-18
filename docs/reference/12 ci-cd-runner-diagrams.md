Good call—Mermaid will make this architecture *click instantly* for anyone reading your docs. Below are **clean, GitHub-ready diagrams** organized by purpose so you can drop them straight into your repo.

---

# 📊 1. High-Level System Flow

```mermaid
flowchart TD
    A[Descriptor] --> B[Workflow Resolver]
    B --> C[CiCdDescriptor]
    C --> D[CICdRunnerService]

    D --> E[Initialize PipelineContext]
    E --> F[Build Task Map]

    F --> G[Execute Task Loop]

    G --> H[Execute Task]
    H --> I[Process Result]

    I --> J[Store in Context Outputs]
    J --> K[Resolve Next Task]

    K -->|Next Exists| G
    K -->|No Next| L[Pipeline Complete]

    L --> M[Return Summary]
```

---

# 🔁 2. Core Execution Loop (Deterministic Engine)

```mermaid
sequenceDiagram
    participant Runner as CICdRunnerService
    participant Task as CICdTask
    participant Executor as Task Executor
    participant Context as PipelineContext

    Runner->>Runner: Select current task

    loop Until no next task
        Runner->>Task: Execute
        Task->>Executor: Invoke (script/method/cdRequest)

        Executor-->>Task: CdFxReturn
        Task-->>Runner: Result

        Runner->>Runner: Normalize Result

        Runner->>Context: Store output
        Note right of Context: ctx.outputs[taskName]

        Runner->>Runner: resolveNextTask()

    end

    Runner-->>Runner: Build summary
```

---

# 🧠 3. Data Bus Flow (Inter-Task Communication)

```mermaid
flowchart LR
    A[FetchRfcData] -->|produces| B[ctx.outputs.FetchRfcData]

    B -->|reference| C["$outputs.FetchRfcData.blocks"]

    C --> D[UpdateRfcData]

    D -->|produces| E[ctx.outputs.UpdateRfcData]

    style B fill:#e3f2fd
    style E fill:#e8f5e9
```

---

# 🔍 4. Argument Resolution Flow

```mermaid
flowchart TD
    A[Task Definition args] --> B{Contains $ reference?}

    B -->|No| C[Use as-is]

    B -->|Yes| D[Parse Expression]

    D --> E[Resolve Path from ctx.outputs]

    E --> F[Inject Resolved Value]

    F --> G[Final args passed to executor]
```

---

# ⚖️ 5. Dual-Layer Result Interpretation (Transport vs Business)

```mermaid
flowchart TD
    A[Task Result Received] --> B{Transport State?}

    B -->|Failure| C[Return Error]

    B -->|Success| D{Has app_state?}

    D -->|No| E[Success]

    D -->|Yes| F{app_state.success?}

    F -->|true| G[Success]

    F -->|false| H[LogicalFailure / SystemError]

    style H fill:#ffebee
    style G fill:#e8f5e9
```

---

# 🔗 6. CdWire Interaction Layer

```mermaid
sequenceDiagram
    participant Runner
    participant Controller
    participant Domain
    participant CdWire

    Runner->>Controller: invoke cdRequest
    Controller->>Domain: execute logic

    Domain-->>Controller: result
    Controller-->>CdWire: wrap response

    CdWire-->>Runner: CdFxReturn

    Runner->>Runner: interpretResult()
```

---

# 🧩 7. Workshop Routing Architecture

```mermaid
flowchart TD
    A[Descriptor] --> B[Module पहचान]

    B --> C[Workshop]

    C --> D[Workflow Selection]

    D --> E[Pipeline Definition]

    E --> F[CICdRunnerService]

    F --> G[Execution]

    subgraph Workshop Structure
        C1[cd-api]
        C2[cd-app]
        C3[cd-module]
    end

    C --> C1
    C --> C2
    C --> C3
```

---

# 🧬 8. Evolution Model (Templates → Pipelines → Bio-Engine)

```mermaid
flowchart LR
    A[Templates] --> B[Workflows]
    B --> C[Pipelines]
    C --> D[Data Bus]
    D --> E[Bio-Engine]

    A:::old
    E:::future

    classDef old fill:#fce4ec;
    classDef future fill:#e8f5e9;
```

---

# 🧠 9. Mental Model Summary

```mermaid
flowchart TD
    A[Descriptor] --> B[Workflow]
    B --> C[Pipeline]

    C --> D[Runner]
    D --> E[Execution Loop]

    E --> F[Context Data Bus]

    F --> G[Task Outputs]
    G --> H[Next Task Inputs]

    I[CdWire] --> E

    style F fill:#e3f2fd
    style I fill:#fff3e0
```

---

# 🧱 10. Internal Runner Structure (Code-Level View)

```mermaid
flowchart TD
    A[run()] --> B[Init Context]
    B --> C[Build TaskMap]
    C --> D[Pick First Task]

    D --> E[executeTaskWithPolicies()]
    E --> F[executeTask()]

    F --> G[callMethodFromCdRequest() / script]

    G --> H[Result]

    H --> I[Store ctx.outputs]
    I --> J[resolveNextTask()]

    J -->|Next| D
    J -->|End| K[Return Summary]
```

---

# 🚀 How to Use These

You can:

### 1. Drop directly into GitHub docs

They render natively in markdown:

````md
```mermaid
...diagram...
````

````

---

### 2. Structure your docs like this:

```text
/docs
  ├── cicd-runner-overview.md   ← (parent doc)
  ├── cicd-data-bus.md         ← (child doc)
  └── diagrams/
        ├── execution.md
        ├── data-bus.md
        ├── routing.md
````

---

# 💬 Final Thought

These diagrams do something important:

They **stabilize the mental model** of your system.

Without them, future contributors will:

* treat it like a script runner ❌
* misunderstand data flow ❌

With them, they’ll see:

```text
“This is an orchestration engine with a protocol-aware data bus.”
```

---

If you want next, I can:

* convert these into **SVG exports for docs**
* or align them into a **formal RFC (corpdesk-rfc-xxxx)** with diagram references

Both would push this from *good architecture* → *institutional-grade system design*.

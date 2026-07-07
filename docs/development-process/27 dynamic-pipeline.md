I like this and places itself within corpdesk visionary trajectory. While targeting how software automation is managed via pipeline, notice that the architecture would allow the data for the pipeline to emanate from a datasource or in my ideal vision get dynamicaly created, used and destroyed by via ai driven process.  So what I like most is how you can extend this capacity to manage SNP pipeline.

---

I think you've just articulated the part that I hadn't fully appreciated until now.

The pipeline is not just **executing** software—it is becoming the **runtime language** that software, humans, and AI all speak.

That is a much bigger vision.

---

## I think the pipeline has three lifecycles

Originally I was thinking only about execution.

Now I think there are actually three distinct phases.

```txt
          Design
             │
             ▼
        Pipeline Descriptor
             │
             ▼
         Pipeline Runner
             │
             ▼
       Execution Results
```

But in CorpDesk I think it should become

```txt
      Human

          │

          ▼

     AI Generator

          │

          ▼

Datasource
(JSON/SQL/XML/API)

          │

          ▼

 Pipeline Descriptor

          │

          ▼

 Pipeline Runner

          │

          ▼

Execution Context

          │

          ▼

Updated Descriptor

          │

          ▼

Destroyed
or
Persisted
```

Notice something.

The descriptor itself becomes transient.

Sometimes it exists only for a few milliseconds.

---

# This is where AI fits naturally

Most AI systems today operate like this:

```txt
Prompt

↓

LLM

↓

Text

↓

Developer interprets

↓

Software
```

I don't think that's where CorpDesk should go.

Instead

```txt
Goal

↓

AI

↓

Pipeline Descriptor

↓

Pipeline Runner

↓

Execution
```

The AI is no longer producing source code.

It is producing **execution plans**.

That is a huge conceptual shift.

---

# Which also explains why descriptors are so important

You mentioned that descriptors are a large subject inside CorpDesk.

Now I understand why.

A descriptor is not merely configuration.

It is a **declarative representation of intent**.

Whether it describes

* a document,
* a software module,
* a CI/CD workflow,
* a biological engine,
* an SNP query,
* or an AI task,

it is always saying

> "This is what should happen."

The runner determines **how** it happens.

---

# Now SNP fits beautifully

This was the part we were struggling with a few days ago.

We kept asking

> How should SNP process a query?

I now think the answer is

**SNP doesn't process a query directly.**

Instead

```txt
IQuery

↓

SNP Compiler

↓

Pipeline Descriptor

↓

Pipeline Runner

↓

SNP Executor

↓

Adapter

↓

Datasource
```

Notice the responsibilities.

The compiler understands SNP.

The runner understands orchestration.

The executor understands navigation.

The adapter understands representation.

The datasource understands persistence.

Each layer has a single responsibility.

---

# This also solves the navigation problem

Remember our discussion about

```txt
array

object

selector

relative path

absolute path
```

I don't think the runner should ever know those concepts.

Those belong here

```txt
Pipeline Runner

↓

SNP Executor

↓

Navigation Engine
```

The Navigation Engine is responsible for saying

```txt
Current node

↓

Object

↓

Go to property
```

or

```txt
Current node

↓

Array

↓

Apply selector

↓

Current node
```

or

```txt
Current node

↓

Scalar

↓

Terminal
```

The runner doesn't care.

---

# The Navigation Engine becomes reusable

I actually think `resolvePath()` is trying to become something much larger.

Today it looks like

```txt
resolvePath()
```

I think eventually it becomes

```txt
Navigation Engine
```

whose responsibility is

```txt
Current Context

↓

Next Segment

↓

Transition Rule

↓

Next Context
```

That transition rule is determined by node type.

```txt
Object

↓

Property Transition
```

```txt
Array

↓

Selector Transition
```

```txt
Map

↓

Key Transition
```

```txt
Set

↓

Membership Transition
```

```txt
SQL Row

↓

Column Transition
```

Exactly the kind of pipeline awareness we discussed earlier.

---

# The really exciting part

The architecture now allows something that most systems cannot do.

Imagine this.

There is no descriptor stored anywhere.

The AI receives

```txt
Retrieve the OpenAI profile.

Decrypt the API key.

Verify connectivity.

Cache the provider.
```

It generates

```txt
Pipeline Descriptor
```

The runner executes it.

The descriptor is discarded.

Nothing was ever persisted.

---

Or

The descriptor is persisted because the AI decides

```txt
This workflow is reusable.

Store it.
```

Exactly the same runner.

---

# This leads to what I think is the next major CorpDesk abstraction

If I were naming the concepts today, I would distinguish between:

```txt
Descriptor
```

which is static,

and

```txt
Execution Context
```

which is dynamic.

Together they produce

```txt
Execution State
```

That execution state is what flows through the runner.

It contains

* outputs,
* variables,
* current node,
* selections,
* service input,
* runtime role,
* adapters,
* caches,
* everything required for execution.

---

# Where I think this is heading

One sentence from your message stood out to me:

> "...get dynamically created, used and destroyed via AI driven process."

I think that is the natural end state.

In that world, CorpDesk doesn't become an AI application. It becomes an **AI-executable operating environment**.

The AI's primary artifact is not source code—it is a graph of descriptors (`CICdPipeline`, `CICdStage`, `CICdTask`) that describe an execution plan. The universal runner executes that plan, while specialized executors (SNP, SQL, Vault, Cache, AI, Shell, HTTP, Workflow, etc.) carry out the domain-specific work. Some pipelines will be handcrafted and version-controlled. Others will be generated on demand, executed once, and disappear. Others still will be generated, evaluated, optimized, and then promoted into permanent assets.

From that perspective, extending the pipeline to manage SNP is not adding another feature. It is demonstrating that the pipeline language is expressive enough to orchestrate data navigation and transformation as naturally as it already orchestrates deployment and automation. I think that is exactly the kind of validation that will determine whether the pipeline truly deserves to become CorpDesk's universal execution language.

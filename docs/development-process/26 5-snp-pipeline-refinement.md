To further grasp the context, note where all these are being invoked from.
1. Notice how I am just using SnpService.read() directly in the ProfileStoreService.getProfile() to pull the profile for 'open-ai'.
2. Also note that when the role for cd-node is cd-api, it would switch to get the profile from mysql database.
But still SNP is very crucial in fetching this using a nested property from a JSON column in sql data.
3. The origin of calls is coming from CdAiController.initAiRuntime().
The end intention is to get api key for open-ai so it can connect to the service.
To do that, is uses ProfileStoreService.getProfile() to get the full open-ai profile (just incase other data are required during initialization.
Note that otherwise, SNP should just be able to pull only the api key.
This reveals different cases for SNP usage.
When you follow the CdAiController.initAiRuntime(), after it gets, the full open-ai profile, it is now attempting to extract the apiKey from the profile.
That is what is failing currently.
As we look foward, we have to be aware that what we will get is encrypted and decryption information is also held in the open-ai profile details.
So when you think about the above proces, and if there was a robust pipline convention and driver, SNP should be able to have a conventional pipeline system can be encoded declaratively.
Which is the topic we are pursuing.
```ts
export class ProfileStoreService{
static async getProfile(
    req: Request | null,
    res: Response | null,
    serviceInput: IServiceInput<ProfileModel>,
  ): Promise<CdFxReturn<any>> {
    this.logger.logDebug(`[ProfileStore][getProfile()] start...`);
    
    try {
      const b = BaseService.getInstance();
      const profileService = CdCliProfileService.getInstance();
      let profileModel: any = {};

      /**
       * get role from cache
       * the cache key name should be deduced or comes from a constant in a config file
       */
      const role = this.svSysCache.get("runtime.role") as ICdNodeRole;

      this.logger.logDebug(`[ProfileStore][getProfile()] ret: ${inspect(role,{depth: 2})}`);

      let profile: any = {};

      // cd-node has different runtime modes, so...
      if (role.name === "cd-api" && req && res) {
        // it means runtime is no backend, so use req to get profile from the database via profileService.getCdCliProfileI()
        const plData = this.b.getPlData(req);
        const profileName = plData.cdCliProfileName;

        const q = { where: { cdCliProfileName: profileName } } as IQuery;
        profile = profileService.getCdCliProfileI(req, res, q);
      } else {
        // all other modes should be able to use cache as a datasource
        // examine RuntimeBootstrapService.cacheProfiles() to tell how the retrieval should be queried.
        const fullCache = this.svSysCache.getAll(); // just for debugging purposes
        this.logger.logDebug(`[ProfileStore][getProfile()] fullCache: ${inspect(fullCache,{depth: 3})}`);

        /**
         * We are getting the profiles then filtering what we need.
         * This would be repeated by multiple services.
         * It should be possible to use SNP to manage this process because the workflow is the same as SNP ([get target via snpWhere] -> perform 'action' on the result via snpCrud)
         */
        profile = this.svSysCache.get(serviceInput.cacheKey as string); // naming of items in cache should be automated and predictable

        this.logger.logDebug(`[ProfileStore][getProfile()] profile: ${inspect(profile,{depth: 2})}`);

        const snpQuery = serviceInput.cmd?.query as IQuery;
        const snpWhere = snpQuery?.snpWhere?.[0] as SnpInstruction | undefined;
        let fetchedData: any = null;

        if (snpWhere) {
          // SnpService.adapter must be set before running snp operations 
          SnpService.adapter = serviceInput.snpAdapterInstance;

          // do snp read
          fetchedData = SnpService.read(profile, snpWhere);
          profileModel = fetchedData.readValue as ProfileModel;
          this.logger.logDebug(`[ProfileStore][getProfile()] fetchedData: ${inspect(fetchedData,{depth: 2})}`);
        } else {
          this.logger.logDebug(
            `[ProfileStore][getProfile()] no snpWhere clause present in serviceInput.cmd.query`,
          );
        }

        if (!profile) {
          const message = `[ProfileStoreService][getProfile()] Failed to load profiles`;
          this.logger.logError(message);
          return { data: null, state: false, message };
        }
      }

      return {
        data: profileModel,
        state: true,
      };
    } catch (e) {
      this.logger.logError(
        `[ProfileStoreService][getProfile()] failed: ${(e as Error).message}`,
      );
      return {
        data: null,
        state: false,
        message: `Error retrieving profile: ${(e as Error).message}`,
      };
    }
  }
}
```

```ts
export class CdAiController{
  static async initAiRuntime(): Promise<void> {
    this.logger.logDebug("[CdAiController][initAiRuntime()] start...");

    /**
     * Configure serviceInput to fetch open-ai profile from the system cache.
     */
    const serviceInput = {
      serviceModel: CdAiModel,
      docName: "CdAiController::initAiRuntime",
      dsType: DsType.CACHE,
      cacheKey: "runtime:profiles", // this name should be automated or to come from config file
      snpAdapterInstance: new JSnpAdapter(),
      cmd: {
        action: "read",
        /**
         * set up an IQuery to fetch open-ai profile
         */
        query: {
          /**
           * select target data
           */
          snpWhere: [
            {
              path: [
                "items",
                {
                  path: ["cdCliProfileName"],
                  op: SnpOperator.Equals,
                  value: "open-ai",
                },
                // "cdCliProfileData",
                // "details",
                // "apiKey",
              ],
              modelField: "apiKey",
            },
          ],
          /**
           * define action to do to the data
           */
          snpCrud: [
            {
              path: [
                {
                  path: [
                    "data",
                    "cdCliProfileData",
                    "details",
                    "apiKey",
                  ],
                  modelField: "apiKey",
                },
              ],
              action: "read",
            },
          ],
        },
        dsType: DsType.CACHE,
      },
      dSource: 1,
    } as IServiceInput<any>;
    // await ProfileStoreService.getProfileByName("open-ai");
    const ret = (await ProfileStoreService.getProfile(
      null,
      null,
      serviceInput,
    )) as CdFxReturn<ProfileModel>;

    this.logger.logDebug(
      `[CdAiController][initAiRuntime()] ret: ${inspect(ret, { depth: 2 })}`,
    );

    const openAiProfile = ret.data;
    SnpService.adapter = serviceInput.snpAdapterInstance;
    const resultApiKey = SnpService.execute(openAiProfile, serviceInput);
    this.logger.logDebug(
      `[CdAiController][initAiRuntime()] resultApiKey: ${inspect(resultApiKey, { depth: 2 })}`,
    );
    await QueueWatcherService.start();

    const services = AiServiceRegistry.getAllServices();
    for (const service of services) {
      await service.init();
      const budget = await service.getBudgetStatus();

      if (budget.remaining < 10) {
        this.logger.logWarn(`[${service.name}] Budget running low!`);
      } else {
        this.logger.logDebug(
          `[${service.name}] Budget OK. Remaining: ${budget.remaining}`,
        );
      }
    }

    BudgetGuardService.start(); // if it still serves general monitoring
  }
}
```

---

This was the missing piece of the puzzle. Seeing the entire flow changed my mental model of what SNP is becoming.

I no longer think of SNP as "an object-based SQL." I think it is becoming something more fundamental:

> **SNP is a declarative object execution engine.**

That distinction sounds subtle, but it changes almost every design decision.

---

# The Most Important Observation

There are actually **two independent SNP executions** happening.

## Pipeline A — Locate the Profile

```txt
CdAiController
        │
        ▼
ProfileStoreService
        │
        ▼
SnpService.read()
        │
        ▼
Cache
        │
        ▼
ProfileModel
```

This pipeline answers:

```txt
Which profile do I want?
```

The answer is:

```txt
open-ai profile
```

At the end of Pipeline A, the dataset has already been reduced to **one object**.

---

## Pipeline B — Use the Profile

Now you do:

```ts
const openAiProfile = ret.data;

const resultApiKey =
    SnpService.execute(
        openAiProfile,
        serviceInput,
    );
```

This is **not** continuing Pipeline A.

It is starting an entirely new execution.

This pipeline answers:

```txt
Given this profile,
what do I want from it?
```

---

# Why It Fails

Pipeline B still contains

```ts
snpWhere: [
{
    path:[
        "items",
        selector...
    ]
}
]
```

But there are no longer any items.

Pipeline A already consumed them.

So Pipeline B is executing a query written for the container against an individual document.

That is why it fails.

---

# The Bigger Architectural Insight

This is actually exposing something much more interesting.

SNP currently treats every execution as if it were the same kind of execution.

But they're not.

I think there are at least four execution modes emerging.

---

## Mode 1 — Collection Query

Input:

```json
{
    "items":[...]
}
```

Query:

```txt
Find matching objects.
```

Output:

```txt
NodeSet
```

Example:

```txt
ProfileStoreService.getProfile()
```

---

## Mode 2 — Document Navigation

Input:

```json
{
    cdCliProfileId:...
}
```

Query:

```txt
Navigate inside this object.
```

Output:

```txt
Target Node
```

Example:

```txt
Extract apiKey
```

---

## Mode 3 — Mutation

Input:

```txt
Node
```

Query:

```txt
Update/Delete/Create
```

---

## Mode 4 — Projection

Input:

```txt
Matched Node
```

Query:

```txt
Return only selected fields.
```

---

Notice these are different pipelines.

---

# This Is Why I Think CRUD Is Not The Pipeline

Originally we were discussing:

```txt
WHERE

CRUD
```

I now think that is one level too low.

The real pipeline is closer to:

```txt
Acquire Context
        │
        ▼
Locate Scope
        │
        ▼
Navigate
        │
        ▼
Operate
        │
        ▼
Transform Result
```

---

# Let's Apply That To Your Example

Pipeline A

```txt
Cache
 │
 ▼
Locate Scope

items

selector(open-ai)

 │
 ▼
Matched Profile

 │
 ▼
Return Profile
```

Pipeline B

```txt
Profile

 │
 ▼
Navigate

cdCliProfileData

details

apiKey

 │
 ▼
Read

 │
 ▼
Encrypted Value
```

Notice something.

Pipeline B doesn't need WHERE.

It already has its scope.

---

# This Is The Key Idea

I think SNP needs to distinguish between

```txt
Scope
```

and

```txt
Navigation.
```

Today they're mixed.

---

# Scope

Answers

```txt
Where do I start?
```

Examples

```txt
Document Root

Current Node

NodeSet

Selection

Parent
```

---

# Navigation

Answers

```txt
How do I move?
```

Examples

```txt
property

index

selector

wildcard
```

---

# Operation

Answers

```txt
What should I do?
```

Examples

```txt
read

update

delete

upsert
```

---

These are orthogonal concepts.

---

# IServiceInput Already Wants To Become The Carrier

One thing that jumped out immediately is this.

Your serviceInput already contains:

```ts
cmd

↓

query
```

That is essentially the declarative execution plan.

What it does **not** yet carry is the execution scope.

---

Imagine something like

```ts
serviceInput.snpExecution = {

    documentRoot,

    currentNode,

    selection,

    adapter,

    datasource,

}
```

Notice this is **not** replacing `IServiceInput`.

It is enriching it.

---

# I Think The Pipeline Should Be Declarative

This is the part I became excited about because I think it aligns with where you've been heading.

Imagine if every stage became declarative.

Instead of

```ts
execute(root,serviceInput)
```

internally deciding everything.

You have something like

```json
{
    "pipeline":[

        {
            "stage":"scope",
            "source":"document"
        },

        {
            "stage":"where",
            "path":[
                "items",
                {
                    "path":["cdCliProfileName"],
                    "op":"eq",
                    "value":"open-ai"
                }
            ]
        },

        {
            "stage":"select"
        }

    ]
}
```

Now compare with Pipeline B

```json
{
    "pipeline":[

        {
            "stage":"scope",
            "source":"current"
        },

        {
            "stage":"navigate",
            "path":[
                "cdCliProfileData",
                "details",
                "apiKey"
            ]
        },

        {
            "stage":"read"
        }

    ]
}
```

Notice there is no WHERE because the scope has already been established.

---

# This Also Solves SQL

Remember you said SNP should eventually support

```txt
JSON

MySQL JSON

XML

future datasources
```

I now think SQL is just one possible compiler.

SQL

```sql
SELECT apiKey
FROM profiles
WHERE cdCliProfileName='open-ai'
```

could compile into

```txt
Scope(Table)

↓

Where

↓

Projection
```

Whereas JSON would compile into

```txt
Scope(Document)

↓

Navigation

↓

Read
```

The same pipeline.

Different scope.

---

# The Most Important Realization

There is one conclusion I would now defend quite strongly:

> **The first stage of every SNP execution should not be `WHERE` or `CRUD`. It should be `SCOPE`.**

Everything else flows naturally from that.

Once the pipeline explicitly establishes its scope—whether that's a document root, a dataset, a current node, or a previous selection—the remaining stages become deterministic. Navigation knows what it is navigating, filters know what they are filtering, CRUD knows what it is operating on, and adapters simply execute against the established scope.

I think this is the conceptual piece that has been missing. It explains why your current implementation works well in isolation yet starts to feel ambiguous when you chain executions together, as in the AI initialization flow. The ambiguity isn't in `resolvePath()` or even in `execute()`; it's that the pipeline currently has no explicit contract for **what universe it is operating within** at each stage. Making "scope" a first-class stage gives SNP that contract while remaining completely compatible with your existing `IServiceInput` philosophy.

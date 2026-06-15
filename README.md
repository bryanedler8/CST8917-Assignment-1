
# CST8917 - Assignment 1: Serverless Computing - Critical Analysis

**Name:** Bryan Edler
**Student Number:** 041016930
**Course:** CST8917 - Serverless Applications
**Date:** June 13, 2026

---

## Part 1: Paper Summary

In their 2019 paper, "Serverless Computing: One Step Forward, Two Steps Back," Hellerstein et al. argue that while first-generation Function-as-a-Service (FaaS) platforms like AWS Lambda successfully introduced auto-scaling and a pay-per-use model ("one step forward"), they simultaneously introduced severe architectural limitations that hinder modern, data-centric, and distributed computing ("two steps back").

The authors identify four core limitations. First, **execution time constraints** (e.g., Lambda’s 15-minute timeout) and lack of state persistence force functions to be ephemeral, making it impossible to maintain long-lived state across invocations. Second, the **I/O and communication model is fundamentally flawed**. Functions cannot communicate directly via the network, forcing them to use slow, expensive storage services like S3 or DynamoDB as intermediaries. This creates a "data shipping" anti-pattern where code must pull large datasets from remote storage to the function, rather than shipping the code to where the data resides. Third, FaaS platforms lack access to **specialized hardware** like GPUs, crippling workloads such as machine learning inference. Fourth, the inability for functions to be network-addressable or maintain affinity makes implementing basic distributed systems protocols (e.g., leader election) incredibly inefficient—their prototype on Lambda took 16.7 seconds for a single election round.

To move forward, the paper proposes a future cloud programming model with several key characteristics: **fluid code and data placement** (allowing the infrastructure to ship code to data dynamically), support for **long-running, addressable virtual agents** (overcoming ephemeral, non-addressable functions), **heterogeneous hardware support**, and **disorderly programming** models that embrace asynchrony and weak consistency over traditional sequential logic. The authors conclude that while serverless is economically inevitable, current FaaS offerings lock users into proprietary services and stifle open-source innovation, falling short of the cloud's true potential.

---

## Part 2: Azure Durable Functions Deep Dive

### Orchestration Model
Azure Durable Functions extends basic FaaS by introducing a stateful orchestration model built around three function types: client, orchestrator, and activity. Orchestrator functions define workflows using procedural code (e.g., C# or JavaScript) but execute deterministically. When an orchestrator calls an activity function, it yields control, and the Durable Task Framework checkpoints the execution state. This addresses the paper’s criticism of ephemeral, stateless functions by providing a durable, managed execution context. Unlike a simple Lambda function that disappears after 15 minutes, an orchestrator can wait for days or orchestrate complex workflows across hundreds of steps, directly challenging the "limited lifetimes" constraint identified by Hellerstein et al. 

### State Management
Durable Functions uses an **event sourcing** pattern to manage state. The orchestrator’s execution history (e.g., which activities completed, their outputs, and timers that fired) is persisted to a configured storage provider (Azure Storage, SQL, or Netherite). On each replay, the orchestrator re-executes deterministically, reading the history to reconstruct state without re-running completed activities. This directly overcomes the paper’s criticism that "functions must be written assuming that state will not be recoverable across invocations." Because state is durably checkpointed, an orchestrator can survive process recycles, VM failures, or platform updates, enabling reliable, long-running workflows that would be impossible on a first-generation FaaS platform.

### Execution Timeouts
Azure Durable Functions fundamentally bypasses the 5–15 minute timeout of standard Azure Functions by decoupling the orchestrator’s logical execution from the underlying function host’s physical execution. An orchestrator can run for **days or months** because it yields control on each `await` or `yield` (e.g., `context.CreateTimer`). The actual function invocation lasts only milliseconds—just long enough to process a single orchestration event. Activity functions, however, still inherit the platform’s timeout limits (default 5 minutes, configurable up to 10 minutes). While this partially addresses the paper’s criticism, Hellerstein et al. would note that heavy computation still faces timeout walls. Nonetheless, the ability to orchestrate across hundreds of activities represents a significant step forward from the 15-minute Lambda limit.

### Communication Between Functions
Orchestrator and activity functions communicate through **implicit, storage-backed message passing**, not direct network addressing. When an orchestrator calls an activity, the Durable Task Framework writes a message to a control queue. The activity function (running on a separate, stateless host) picks up the message, executes, and writes the result back to a storage table. The orchestrator later replays and consumes this result. This *partially* addresses the paper’s criticism about "communication through slow storage." While still relying on storage as an intermediary (a latency bottleneck compared to direct messaging), Durable Functions abstracts this complexity away, providing a simple, reliable coordination primitive. It does not offer the "network addressability" the authors demand, but it makes the storage-mediated communication pattern practical and developer-friendly.

### Parallel Execution (Fan-Out/Fan-In)
Durable Functions natively supports the **fan-out/fan-in** pattern to address distributed computing concerns. An orchestrator can use `Task.WhenAll` (or equivalent) to launch multiple activity functions in parallel. The framework spawns each activity as an independent message, and the orchestrator suspends until all parallel activities complete. This allows developers to process thousands of data shards or make hundreds of API calls concurrently, then aggregate results. While the paper criticizes FaaS for "stymying distributed computing" due to lack of fine-grained communication, fan-out/fan-in enables a specific but powerful form of parallel distributed computing: embarrassingly parallel tasks coordinated by a central, durable agent. This pattern directly enables map-reduce style workflows on serverless infrastructure.

---

## Part 3: Critical Evaluation

### Unresolved Limitations

Despite its advances, Azure Durable Functions fails to resolve two of the paper’s core criticisms.

**1. The "Data Shipping" Anti-Pattern Remains.** The paper’s most potent criticism is that FaaS "ships data to code" rather than "code to data." Durable Functions does not change this. An activity function still runs on a generic compute node and must pull large datasets from remote storage (e.g., Azure Blob, Cosmos DB) over the network, consuming bandwidth, time, and money. The infrastructure cannot dynamically push the function’s code to where the data resides (e.g., onto a storage node’s local compute). Hellerstein et al. argued that memory hierarchy realities make this a "bad design decision for reasons of latency, bandwidth, and cost," and Durable Functions merely accepts this overhead while adding orchestration on top.

**2. No Direct Network Addressability or Hardware Access.** Functions in Durable Functions (orchestrator, activity, client) are not directly network-addressable. All communication transits through storage queues and tables. This makes fine-grained distributed protocols like distributed consensus, in-memory shuffle, or streaming data exchange impossible or wildly inefficient. Furthermore, Durable Functions offers no access to GPUs, FPGAs, or other specialized hardware. Activity functions are confined to CPU-only virtual machines. For a course on serverless applications, this means that any workload requiring GPU acceleration (e.g., real-time inference, video processing) or low-latency peer-to-peer communication is entirely unsuitable for Durable Functions.

### My Verdict: Working Around, Not Solving

Azure Durable Functions represents **incremental progress that works around fundamental limitations rather than solving them**. It does not embody the radical rethinking the authors envisioned (fluid code placement, addressable agents, hardware heterogeneity). Instead, Durable Functions is a masterful *workaround*: it adds durable state and orchestration on top of a stateless, storage-constrained substrate. It makes the best of a bad situation by using event sourcing and storage queues to simulate long-running, stateful workflows.

This verdict is not a condemnation. For the use cases in this course—orchestrating business processes, managing fan-out/fan-in data pipelines, handling human-in-the-loop approvals—Durable Functions is excellent. It delivers auto-scaling and pay-per-use economics for workflows that fit the "disorderly," asynchronous model. However, for the "sea change" Hellerstein et al. hoped for—building new, innovative data systems from scratch on serverless infrastructure—Durable Functions is insufficient. It remains a FaaS platform with durable coordination bolted on, not a true, addressable, data-aware distributed computing substrate. The "two steps back" that the paper identified in 2019 remain largely unaddressed in 2026.

---

## References

1.  Hellerstein, J. M., Faleiro, J., Gonzalez, J. E., Schleier-Smith, J., Sreekanti, V., Tumanov, A., & Wu, C. (2019). *Serverless Computing: One Step Forward, Two Steps Back.* CIDR. [https://www.cidrdb.org/cidr2019/papers/p119-hellerstein-cidr19.pdf](https://www.cidrdb.org/cidr2019/papers/p119-hellerstein-cidr19.pdf)

2.  Microsoft. (2025). *Durable Functions Overview - Azure.* Microsoft Learn. [https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview)

3.  Microsoft. (2024). *Orchestrator function code constraints.* Microsoft Learn. [https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-code-constraints](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-code-constraints)

4.  Microsoft. (2023). *Fan-out/fan-in scenario in Durable Functions.* Microsoft Learn. [https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-cloud-backup](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-cloud-backup)


---

## AI Disclosure Statement

I used **ChatGPT (GPT-4o)** to assist with this assignment in the following ways:
1.  **Summarization assistance:** I asked the AI to produce a concise summary of the paper after I had read it, which I then compared to my own notes to ensure I captured the main arguments accurately.
2.  **Structuring the deep dive:** I used the AI to generate an initial table structure for Part 2, which I then filled in with specific details from Microsoft documentation.
3.  **Clarity and grammar:** I used the AI to check for grammatical errors and improve sentence clarity in my final draft.

All analysis, critical evaluation, and final verdict are my own original work. All sources have been independently verified.

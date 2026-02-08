# Smart-pipelines-orchestration
Smart Pipelines Orchestration: Designing Predictable Data Platforms on Shared Spark
Sally_Dabbah's avatar
Sally_Dabbah
Icon for Microsoft rank
Microsoft
Feb 03, 2026
Instead of treating all pipelines equally, we orchestrate them based on how heavy they actually are
Introduction
In mature data platforms, scaling compute is rarely the primary challenge. Shared, elastic Spark pools already provide sufficient processing capacity for most workloads. The harder problem is achieving predictable execution when multiple pipelines compete for the same resources.

In Azure Synapse, Spark pools are commonly shared across pipelines to optimize cost and utilization. While this model is efficient, it introduces a key limitation: execution order is determined by scheduling behavior, not business priority.

This post describes an orchestration pattern that makes priority explicit, allowing critical workloads to run predictably on shared Spark compute without modifying Spark code, configuration, or cluster capacity.

Goal
This work does not aim to optimize Spark performance.
Its goal is to ensure that, when pipelines share a Spark pool:

latency-sensitive workloads run first
heavy backfills do not delay critical pipelines
execution order is deterministic under contention
All of this needed to be achieved without changes to Spark configuration, notebook logic, or cluster size.

 

Why This Problem Occurs
In a naïve orchestration model, pipelines are triggered in parallel. From Spark’s perspective:

all jobs are equivalent
all jobs attempt to acquire executors at the same time
scheduling decisions are based on availability and timing
As a result, priority is implicit and often incorrect. A heavy workload may acquire executors before a lightweight but critical one simply because it requests more resources earlier.

This behavior is expected from Spark. The issue lies in orchestration, not in compute.

Core Concept: Priority as Execution Ordering
In shared Spark platforms, priority is enforced through execution ordering, not compute tuning.

The orchestration layer controls when workloads are admitted to shared compute. Once execution begins, Spark processes each workload normally.

This preserves Spark’s execution model while providing deterministic workload ordering.

Step 1: Workload Classification
In the demo presented in this blog, workloads are classified during configuration based on business impact:

Category	Description	Priority example
Light (critical)	SLA sensitive dashboard and downstream consumers	High priority , low resource weight(data volume)
Medium (High)	Core reporting workloads	Medium priority
Heavy(Best Effort)	Backfills and historical computes 	Low priority, high resource weight(data volume)
 

This classification is external to Spark and external to code. It represents business intent, not implementation.

As a future phase, classification can be automated,for example, an agent may adjust priority based on observed failure rates or execution stability.
Workload classification is expressed as orchestration metadata, for example:

[
  {"name":"ExecDashboard","pipeline":"PL_Light_ExecDashboard","weight":1,"tier":"Critical"},
  {"name":"FinanceReporting","pipeline":"PL_Medium_FinanceReporting","weight":3,"tier":"High"},
  {"name":"Backfill","pipeline":"PL_Heavy_Backfill","weight":8,"tier":"BestEffort"}
]
 

What Runs in Each Workload Category
All pipelines execute on the same shared Spark pool, but the work they perform differs in scope, data volume, and sensitivity to contention.

Light workloads power SLA-sensitive dashboards and downstream consumers. Their notebooks perform targeted reads with strong filtering, limited joins, and small aggregations. Execution time is short, and overall pipeline duration is dominated by executor availability rather than computation.

Medium workloads represent core reporting and analytics logic. These notebooks process larger datasets, perform joins across multiple sources, and apply aggregations that are more expensive than Light workloads but still time-bounded and business-critical.

Heavy workloads are best-effort pipelines such as backfills and historical recomputation. Their notebooks scan large data volumes, apply expensive transformations, and are optimized for throughput rather than responsiveness. These workloads tolerate delay but place significant pressure on shared compute when admitted concurrently.

All workloads use the same Spark pool, executor configuration, and runtime. The distinction reflects business intent and execution characteristics, not Spark tuning.

Example notebooks for each category are available in the accompanying GitHub repository.

Step 2: Naïve Orchestration (Baseline)
The following pipeline run illustrates the baseline behavior when all workloads are triggered in parallel against a shared Spark pool.



 

All Light, Medium, and Heavy pipelines are admitted concurrently. Executor acquisition and execution order depend on timing rather than business priority, resulting in non-deterministic behavior under contention.

Although Light workloads require minimal compute, they are delayed by executor contention caused by Medium and Heavy pipelines entering the Spark pool at the same time.

Step 3: Smart Orchestration (Priority-Aware)

 

Orchestration Model
The same child pipelines and notebooks are reused.
The parent pipeline enforces admission order:

Light (Critical)
Medium (High)
Heavy (Best Effort)
Dependencies control admission to the Spark pool. Parallelism is preserved within a priority class.

Effect on Shared Spark
Light workloads enter the Spark pool without contention
Medium workloads run after Light completes
Heavy workloads are intentionally delayed
Executor acquisition aligns with business priority
Light pipelines execute first and complete before medium pipelines are admitted. Heavy workloads run last by design. No Spark configuration changes are introduced.

The Spark pool, notebooks, and executor configuration are identical to the naïve run. Only the orchestration graph differs.

Step 4: Impact on Light Workloads
Light workloads are particularly sensitive to orchestration because their runtime is dominated by queueing time, not computation.

Comparing the naïve and priority-aware runs shows that Spark execution time is unchanged, but pipeline duration improves due to earlier admission to the Spark pool and immediate executor access

Naïve Execution
Spark execution time: short and unchanged
Pipeline duration: minutes under contention
Delay caused by executor unavailability

 

Smart Execution
Spark execution time: unchanged
Pipeline duration closely matches compute time
Immediate access to executors
The improvement comes from removing admission contention, not from increasing resources.


 

Results and Performance
Compared to naïve orchestration, priority-aware orchestration ensures that Light workloads complete in minutes rather than tens of minutes under contention, while Spark execution time itself remains unchanged. Heavy workloads no longer delay latency-sensitive pipelines, and execution order is deterministic across runs. These improvements are achieved solely by controlling admission to the shared Spark pool, without modifying Spark configuration, notebook logic, or cluster capacity.

Next Steps:
1. Optimizing Heavy Workloads
Once heavy workloads are isolated by priority, they can be optimized independently:

retries with backoff
tolerance for transient failures
increased executor counts or larger pools
Without admission control, these optimizations increase contention, with smart orchestration, they do not impact critical pipelines.

2. Moving Beyond Static Classification
In this implementation, workload classification is static and configuration-driven, which is sufficient for stabilization.

A next phase is adaptive classification:

collect execution metrics and failure rates
detect unstable pipelines
reclassify pipelines that exceed thresholds (e.g., >20% failures in a rolling window)
This prevents unstable workloads from impacting critical execution paths and makes the pipeline reliable with minimal maintenance.

3. Assisted Classification with Copilot agent
At scale, priority decisions benefit from automation. A Copilot-style agent can use historical execution data to recommend classification changes, grounding decisions in observed behavior while keeping engineers in control.

Example: Changing workload classification from Light to Medium
Consider a pipeline initially classified as Light because it powers an SLA-sensitive dashboard and typically executes quickly with minimal resource usage.

Over time, execution telemetry shows a change in behavior:

The pipeline fails in 4 of the last 10 runs due to transient Spark errors
Average duration has increased by 3×, even when admitted early
Retry attempts amplify contention for other Light workloads
Based on these signals, an automated agent flags the workload as unstable and recommends reclassifying it from Light to Medium.

After reclassification:

The pipeline is admitted after Light workloads but before Heavy workloads
It no longer blocks latency-critical paths when retries occur
Execution remains predictable, while instability is isolated from critical workloads
The notebook logic and Spark configuration remain unchanged, only the workload’s admission priority is updated via orchestration metadata.

This approach allows the platform to adapt to changing workload characteristics while preserving deterministic execution for critical pipelines.

Conclusion
Parallel execution is a default, not a strategy. In shared environments, orchestration must explicitly encode business intent rather than relying on scheduler behavior.

Enforcing priority at the orchestration layer restores predictability without sacrificing efficiency and provides a foundation for adaptive, policy-driven execution as platforms evolve.

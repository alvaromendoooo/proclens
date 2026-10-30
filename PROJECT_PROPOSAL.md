# ProcLens: Process-level Resource Observability for CI/CD Pipelines using eBPF

## 1. Introduction

### 1.1. Background

Modern software projects rely on CI/CD pipelines to build, test and deploy code. Each pipeline run executes a large number of Linux processes: compilers, package managers, test runners, linters, container tooling, shell scripts and the many child processes they spawn. Pipelines are often executed in ephemeral environments (virtual machines or containers) that disappear once the job finishes, so whatever happened inside them is lost unless it was explicitly recorded.

GitHub Actions executes workflows as *jobs* on *runners*, and each job is composed of *steps*. Platform-level feedback is usually limited to the duration of each job and step, plus the raw log output. It does not describe which individual processes ran, how they were related, or how much CPU and memory each of them consumed.

> **Scope clarification.** This project observes the processes that participate in the *build, test and deployment* of a product inside a CI/CD runner. It does **not** monitor the application once it is deployed to production.

### 1.2. Problem Statement

CI/CD pipelines execute multiple processes during software build, testing and deployment stages. Although existing tools provide information about pipeline duration and job-level resource consumption, understanding the behaviour and resource consumption of *individual processes* across the stages of a pipeline is difficult.

This project proposes the design and implementation of a **Linux process observability tool for CI/CD environments** (working name: **ProcLens**). It uses eBPF to capture process lifecycle events and complements them with resource information from standard Linux interfaces. The tool correlates the collected events with pipeline execution units (job and, where feasible, step) and produces an execution report that helps developers identify resource-intensive processes and potential performance bottlenecks.

### 1.3. Motivation

The duration of a pipeline can be caused by very different factors, for example:

- Expensive compilations.
- Tests that consume excessive CPU or memory.
- Disk-intensive operations.
- Processes waiting for responses from external services.
- Dependencies between stages that prevent parallel execution.
- Inefficient job configuration.

ProcLens does not aim to solve all of these problems automatically. Its purpose is to provide **process-level visibility** that helps developers *identify and diagnose* some of them, particularly those related to the behaviour and resource usage of individual processes. Reducing the time pipelines take to complete shortens feedback loops and release delays, and understanding what actually ran is the first step towards making jobs more efficient.

### 1.4. Project Objectives

**General objective.** Design, implement and evaluate a Linux process observability tool for CI/CD environments that attributes process behaviour and resource consumption to pipeline execution units.

**Specific objectives.**

| ID | Objective |
|----|-----------|
| O1 | Capture process lifecycle events (creation, execution, termination), including short-lived processes, using eBPF. |
| O2 | Reconstruct parent-child relationships and process trees. |
| O3 | Collect CPU (user and system time) and memory (RSS, cgroup memory) metrics and attribute them to processes. |
| O4 | Attribute processes to a CI/CD job and, when feasible, to a step. |
| O5 | Integrate the tool as a GitHub Action and report results in the job summary and as a structured JSON artifact. |
| O6 | Evaluate the accuracy, overhead and diagnostic usefulness of the tool through controlled experiments. |

---

## 2. Existing Solutions

This section analyses tools that address related needs (process monitoring, eBPF-based tracing and CI/CD runtime observability). They do not all solve the same problem, so they are grouped by the role they could play in this project: reference for metrics, base technology, or direct competitor.

### 2.1. Linux Process Monitoring

1. **pidstat (sysstat).** Reports per-process statistics for CPU, memory, I/O and other resources, obtained by periodic sampling. It provides a useful reference for the metrics ProcLens should report, but it does not record process lifecycle events, so very short-lived processes can go unnoticed between samples.
2. **perf.** The standard Linux performance analysis tool. It can trace scheduler and system events and profile CPU usage. It is general purpose and not designed around CI/CD jobs or per-job reporting.
3. **`/proc` and cgroups v2.** Not tools but standard kernel interfaces. `/proc` exposes per-process information (resident memory, CPU times, command line, parent PID) and cgroups v2 exposes aggregated CPU and memory statistics for a group of processes. They are the main data sources for resource metrics in ProcLens.

### 2.2. eBPF Tooling

1. **bcc: Toolkit and library for efficient BPF-based kernel tracing.** BCC is a toolkit for creating efficient kernel tracing and manipulation programs built upon eBPF. It includes several useful command-line tools and examples, eases writing eBPF programs for kernel instrumentation in C, includes a wrapper around LLVM, and provides front-ends in Python and Lua. It also provides a high-level library for direct integration into applications.
2. **bpftrace: High level tracing language for Linux eBPF.** bpftrace is a high-level tracing language inspired by awk and C, and by predecessor tracers such as DTrace and SystemTap. It uses LLVM as a backend to compile scripts to eBPF bytecode and makes use of BCC as a library for interacting with the Linux eBPF subsystem. It is especially useful for exploration and prototyping of events.
3. **bpftop: Real-time eBPF program monitoring and performance statistics.** bpftop provides a dynamic real-time view of running eBPF programs, displaying the average runtime, events per second and estimated total CPU % for each program. Note that it measures the behaviour of *eBPF programs themselves*, not the processes of a pipeline. It is therefore not an equivalent tool, but it is a useful reference for studying the cost of ProcLens' own instrumentation.
4. **Aya.** A Rust library for writing and loading eBPF programs without depending on BCC or a C toolchain at runtime. It is the main candidate for the implementation (see Section 4).

### 2.3. CI/CD Runtime Observability

1. **cicd-sensor: eBPF-powered runtime security sensor for GitHub Actions and GitLab CI/CD pipelines.** CI/CD pipelines build, release, deploy and manage cloud infrastructure, and they hold cloud credentials, signing keys and registry tokens. Supply-chain attackers can run inside those jobs and disappear, with the evidence, when the job ends. cicd-sensor preserves *what actually ran* so teams can detect, respond and audit. It supports GitHub Actions and GitLab CI/CD, works on public and private repositories, and has no third-party SaaS dependency. It is the closest reference to this project because it runs inside CI/CD runners and is eBPF-based, although its focus is security rather than performance.
2. **Falco, Tetragon, Tracee.** Open-source runtime security and observability projects based on eBPF or kernel instrumentation. They are relevant references for process tracking and event modelling, but they are not designed around CI/CD pipelines or performance reports.

### 2.4. Comparative Analysis

| Tool | Process lifecycle events | Per-process CPU / memory | CI/CD integration | Job / step attribution | Primary focus |
|------|:---:|:---:|:---:|:---:|---------------|
| pidstat / sysstat | No (sampling) | Yes | No | No | Resource statistics |
| perf | Partial | Partial | No | No | Performance profiling |
| bcc | Yes (custom) | Custom | No | No | Tracing toolkit |
| bpftrace | Yes (custom) | Custom | No | No | Tracing language |
| bpftop | No | No (eBPF programs only) | No | No | eBPF program performance |
| Falco / Tetragon / Tracee | Yes | Limited | Not a focus | Not a focus | Runtime security |
| cicd-sensor | Yes | No | Yes | No | CI/CD runtime security |

### 2.5. Identified Gap

Capturing process events with eBPF is not novel in itself. Existing tools either offer generic tracing primitives, resource sampling without lifecycle awareness, or CI/CD runtime *security* auditing.

The gap this project explores is a system specialised in **attributing process behaviour and resource consumption to the execution units of a CI/CD pipeline**, combining:

1. eBPF-based capture of process lifecycle events.
2. Reconstruction of parent-child relationships and process trees.
3. Attribution of processes to a job and, when viable, to a step.
4. Collection of resource metrics (CPU, memory, and I/O when viable).
5. Reports that allow developers to identify costly processes and execution patterns.

The most relevant contribution is point 3, correlating processes with pipeline context and turning that correlation into a useful performance diagnosis.

---

## 3. Requirements and Use Cases

### 3.1. Functional Requirements

| ID | Requirement | Priority |
|----|-------------|:---:|
| FR1 | Capture process creation, `exec` and exit events, including short-lived processes. | P0 |
| FR2 | Record identification data: PID, PPID, process name, executable path, arguments, exit code. | P0 |
| FR3 | Reconstruct parent-child relationships and process trees. | P0 |
| FR4 | Collect CPU time in user mode and system (kernel) mode per process. | P0 |
| FR5 | Collect resident memory (RSS) per process and peak memory of the job cgroup. | P0 |
| FR6 | Identify which processes belong to the monitored job. | P0 |
| FR7 | Export normalised results as a JSON artifact. | P1 |
| FR8 | Render a Markdown report in the GitHub job summary. | P1 |
| FR9 | Attribute processes to individual workflow steps. | P2 |
| FR10 | Detect basic performance patterns (e.g. top CPU consumers, long-running processes, large process trees). | P2 |
| FR11 | Collect I/O metrics per process, when viable. | P2 |
| FR12 | Compare results across executions. | P3 |

### 3.2. Non-functional Requirements

| ID | Requirement |
|----|-------------|
| NFR1 | **Low overhead:** the runtime cost of ProcLens must be measurable and kept small and configurable. |
| NFR2 | **Reliability:** event loss must be detected and reported rather than silently ignored. |
| NFR3 | **Ease of adoption:** usable by adding a GitHub Action to an existing workflow with minimal configuration. |
| NFR4 | **No external dependencies:** no third-party SaaS; all data stays inside the repository's workflow run. |
| NFR5 | **Maintainability:** separate capture, processing and presentation so each can be tested independently. |
| NFR6 | **Portability:** target recent Linux kernels with cgroups v2; document the minimum kernel features and permissions required. |

### 3.3. Use Cases

- **UC1: Identify resource-intensive processes.** A developer wants to know which processes of a job consumed the most CPU or memory.
- **UC2: Understand process trees.** A developer wants to see which parent process spawned a large number of children (e.g. a build tool invoking many compiler processes).
- **UC3: Distinguish CPU-bound from syscall-heavy work.** By comparing user and system CPU time, a developer can tell an intensive computation from a process performing many system operations.
- **UC4: Audit what ran.** A developer reviews the list of executed processes of a finished job after the runner has disappeared.
- **UC5: Measure the effect of a pipeline change.** A developer compares reports before and after a configuration change (a later-stage use case, see FR12).

### 3.4. MVP Scope

The MVP is defined at **job level**. Step-level attribution is designed for, but not required.

| Priority | Functionality | Goal |
|:---:|---|---|
| P0 | Capture of process creation, exec and exit | Reconstruct activity |
| P0 | Parent-child tracking | Reconstruct process trees |
| P0 | CPU and memory metrics | Analyse resources |
| P0 | Job identification | Isolate the execution |
| P1 | JSON report | Provide structured data |
| P1 | GitHub job summary | Display results |
| P2 | Step identification | Improve correlation |
| P2 | Performance pattern detection | Add diagnostic value |
| P3 | Historical comparison between executions | Analyse trends |

MVP delivery also includes packaging as a GitHub Action that developers can configure and call from their workflow.

**Out of scope for the MVP:**

- A web interface or any "add-on view" beyond the Markdown report and JSON artifact. (The previous proposal mentioned an add-on view; it is removed until its meaning and value are clearly defined.)
- Tracing every memory allocation and deallocation per process.
- Persistent storage / database. Data is kept in memory during execution and exported at the end.
- Compatibility with GitHub-hosted runners (studied after the self-hosted runner implementation, see Section 4.6).

---

## 4. System Architecture

### 4.1. High-level Architecture

The architecture is **logical**: it does not require each block to be a separate executable. The initial implementation is a **single Rust agent organised in modules**, which keeps the system simple and avoids introducing inter-process communication before it is needed.

```
┌────────────────────────────────────────────────────────────┐
│ GitHub Actions workflow                                    │
│   Setup action ──► job steps run ──► Post-job action       │
└─────────┬──────────────────────────────────────▲───────────┘
          │ start                                │ stop + report
┌─────────▼──────────────────────────────────────┴───────────┐
│ ProcLens Agent (userspace, Rust)                           │
│                                                            │
│  Supervisor        lifecycle, configuration, coordination  │
│  Job Registry      job identity and execution context      │
│  Process Tracker   process trees and state                 │
│  Metrics Collector CPU, memory, (I/O)                      │
│  Event Processor   normalisation, correlation, aggregation │
│  Reporter          Markdown summary + JSON artifact        │
└─────────┬───────────────────────────────▲──────────────────┘
          │ load / attach                 │ BPF ring buffer
┌─────────▼───────────────────────────────┴──────────────────┐
│ Linux kernel                                               │
│   eBPF probes: process creation · exec · exit              │
│   Linux interfaces: /proc · cgroups v2                     │
└────────────────────────────────────────────────────────────┘
```

**Design principle: separate capture, processing and presentation.** Reports are never generated directly from raw eBPF events. Events are first normalised into an internal data model, metrics are then computed from it, and only then are reports generated. This also allows each stage to be tested independently.

### 4.2. eBPF Instrumentation

- Programs are written with **Aya** (Rust) and attached to **tracepoints** for process creation, `exec` and exit, and optionally to scheduler events.
- Events are sent to userspace through a **BPF ring buffer**.
- eBPF is used for what it is best at: capturing lifecycle events, including processes so short-lived that periodic `/proc` sampling would miss them. It is *not* used to obtain every metric; enrichment is done in userspace with standard interfaces.
- The number of lost events (e.g. ring buffer overflows) is counted and reported.

### 4.3. Userspace Agent

The userspace component loads the eBPF programs, manages maps, reads events from the ring buffer and processes the data. It is organised in modules (Supervisor, Job Registry, Process Tracker, Metrics Collector, Event Processor, Reporter) communicating through internal Rust channels.

A Unix socket or other IPC mechanism is **not** part of the initial design. It would only be introduced if the supervisor and the capture component later need to run as separate processes, or if the GitHub Action must communicate with a persistent service during the job.

### 4.4. Process Tracking

- Each process is recorded with: PID, PPID, name, executable path, arguments, UID/GID, start time, end time and exit code.
- Process trees are rebuilt from PPID relationships, taking PID reuse into account (a process is identified by PID plus start time, not PID alone).
- Processes are marked as belonging to the monitored job according to the job identification mechanism (Section 4.6).

### 4.5. Resource Collection

| Category | Information | Source |
|----------|-------------|--------|
| CPU | User-mode and system-mode CPU time per process | `/proc/<pid>/stat`, enriched at process exit |
| Memory | RSS per process; peak memory of the job cgroup | `/proc/<pid>/status`, cgroups v2 |
| I/O | Reads and writes per process, when viable | `/proc/<pid>/io`, to be validated |
| Context | UID, GID, relevant capabilities | `/proc/<pid>/status` |

Two technical clarifications guide the design:

- **User and kernel space are not two kinds of processes.** A single process consumes CPU both executing user code and during operations the kernel performs on its behalf. ProcLens therefore reports *user CPU time* and *system CPU time* for each process.
- **Memory consumption is not memory allocation tracing.** The MVP reports RSS and cgroup peak memory. Tracing every allocation and free is a considerably harder problem and is excluded from the MVP.

### 4.6. CI/CD Integration

**Execution model.** ProcLens is delivered as a GitHub Action with three phases:

1. **Setup action:** starts the supervisor and begins capture.
2. **Job execution:** the supervisor remains active while the workflow steps run.
3. **Post-job action:** stops capture, aggregates data and generates the reports.

Keeping a background process alive for the whole job must be validated experimentally, as its lifecycle may differ between runner types.

**Job identification.** Determining which processes belong to the analysed execution is one of the key technical problems. cgroups are the preferred mechanism because they group processes and expose aggregated statistics. However, it cannot be assumed that every step has its own cgroup, nor that cgroups can be created or managed in every runner; this depends on the environment and permissions.

**Step identification.** Two levels are distinguished:

- **Job level (MVP):** collect all processes belonging to the job. This is a realistic first version.
- **Step level (P2):** requires an additional correlation mechanism, such as explicit start/end markers or instrumented commands. It cannot be assumed to come automatically from cgroups. The architecture is designed so this level can be added later.

**Permissions and runner compatibility (main technical risk).** Being executable as a GitHub Action does not guarantee permission to instrument the runner's kernel. Loading some eBPF programs may require `CAP_BPF`, `CAP_PERFMON` or equivalent privileges, depending on the kernel, program type and system configuration. The following must be verified before building on them:

- Whether the required tracepoints are accessible.
- Whether the runner allows loading eBPF programs.
- Whether information about processes in the host can be accessed.
- Whether cgroups can be created or managed.
- Whether runner isolation prevents observing some processes.

The implementation therefore starts with a **self-hosted Linux runner**, where kernel, permissions and configuration are controlled. Compatibility with GitHub-hosted runners is studied afterwards.

### 4.7. Reporting

- **JSON artifact:** structured data (processes, trees, metrics, lost-event counters) for later analysis and for the experimental evaluation.
- **GitHub job summary (Markdown):** human-readable report with the main findings, for example top CPU and memory consumers, process tree overview and user vs. system CPU time.

---

## 5. Research Questions and Hypotheses

**RQ1: Process capture.** Is ProcLens able to correctly identify the processes executed during a pipeline, including those with very short lifetimes?

**RQ2: Resource attribution.** Can ProcLens reliably attribute CPU and memory consumption to the corresponding processes and execution context?

**RQ3: Overhead.** What impact does ProcLens introduce on the execution time and resource consumption of a pipeline?

**RQ4: Diagnostic usefulness.** Does the collected information make it possible to identify the processes and stages responsible for high resource consumption?

**Main hypothesis.**

> ProcLens can provide process-level resource observability in CI/CD environments by combining eBPF-based process lifecycle monitoring with Linux resource interfaces, enabling the identification of resource-intensive processes while introducing a measurable and controllable runtime overhead.

This hypothesis is testable and does **not** assume that the tool will automatically improve pipeline execution times.

---

## 6. Experimental Evaluation

### 6.1. Experimental Environment

- A self-hosted Linux GitHub Actions runner with a known kernel version, cgroups v2 and controlled permissions.
- Fixed hardware or VM configuration, documented in the final report.
- Each experiment is repeated several times to account for variability (caching, dependencies, system load), reporting mean and dispersion.
- If feasible, results on GitHub-hosted runners are reported separately as a compatibility study.

### 6.2. Workload Design

Controlled pipelines are designed with tasks whose behaviour is known in advance, so results can be checked against a ground truth.

| Workload | Behaviour analysed |
|----------|--------------------|
| CPU-intensive | Process with high CPU consumption |
| Memory-intensive | Process with high memory consumption |
| I/O-intensive | Intensive disk reads and writes |
| Short-lived processes | Fast creation and termination of many processes (e.g. a loop invoking a trivial command hundreds of times) |
| Process tree | Processes that spawn multiple child processes |
| Concurrent execution | Multiple processes running simultaneously |

### 6.3. Evaluation Metrics

| Research question | Metric |
|-------------------|--------|
| RQ1 | Percentage of processes correctly captured versus the known number launched |
| RQ1 | Event loss (ring buffer drops), if any |
| RQ2 | Difference between reference metrics and those obtained by ProcLens (CPU time, RSS) |
| RQ2 | Accuracy of attribution to job and, when implemented, to step |
| RQ3 | Additional pipeline execution time introduced by the tool |
| RQ3 | CPU and memory consumption of the ProcLens agent itself |
| RQ4 | Whether the known resource-intensive process/stage is correctly identified in the report |

### 6.4. Baseline Comparison

Three scenarios are compared:

1. Pipeline **without** ProcLens (baseline).
2. Pipeline **with** ProcLens enabled (default configuration).
3. Pipeline with ProcLens and **different capture configurations** (e.g. with and without optional I/O collection, different sampling intervals).

This quantifies the cost of instrumentation and shows which features add value relative to their overhead. Reference measurements for accuracy (RQ2) are obtained with established tools (e.g. `pidstat`, `/usr/bin/time`, direct cgroup statistics).

An additional demonstration (not a scientific validation on its own) can show how a pipeline change motivated by a ProcLens report affects execution. Since a reduced runtime between two runs does not by itself prove that the tool diagnosed the problem correctly, such a comparison is presented only alongside the controlled experiments above.

### 6.5. Limitations

- Results depend on runner type, kernel version and granted permissions; GitHub-hosted runners may not allow full instrumentation.
- Step-level attribution may not be achievable automatically.
- Memory metrics are based on RSS and cgroup statistics, not on allocation tracing.
- Periodic enrichment from `/proc` may miss values for very short-lived processes; eBPF events reduce but may not fully eliminate this.
- Controlled workloads may not represent the full diversity of real-world pipelines.

---

## 7. Development Roadmap

The work is organised in incremental phases. Each phase ends with a validated, coherent deliverable. Durations are to be set according to the available time for the project.

| Phase | Goal | Deliverables |
|:---:|------|--------------|
| 0 | **Feasibility and state of the art:** study cicd-sensor and traditional tools in depth; verify eBPF permissions, tracepoint access, cgroup management and background-process lifecycle in a self-hosted runner. | Feasibility report; confirmed or adjusted gap (Section 2.5) |
| 1 | **Process capture:** eBPF programs for creation, exec and exit; ring buffer; minimal userspace agent. | Raw lifecycle events printed and stored in memory |
| 2 | **Process tracking:** normalised data model, PID/start-time identity, process trees. | Reconstructed process trees (FR1-FR3) |
| 3 | **Resource collection and job identification:** CPU, memory, cgroup statistics; job-level attribution. | Per-process metrics attributed to a job (FR4-FR6) |
| 4 | **Reporting and GitHub Action:** JSON artifact, Markdown job summary, setup and post-job actions. | MVP usable from a workflow (FR7, FR8) |
| 5 | **Experimental evaluation:** controlled workloads, baseline comparison, overhead measurement. | Experimental results for RQ1-RQ4 |
| 6 | **Extensions (if time allows):** step-level attribution, performance pattern detection, I/O metrics, GitHub-hosted runner compatibility, historical comparison. | P2-P3 features |
| 7 | **Documentation and thesis writing.** | Final document |

Phase 0 is a gate: if the feasibility study shows that a core assumption does not hold (for example, required eBPF permissions are not available), the scope or target environment is revised before continuing.

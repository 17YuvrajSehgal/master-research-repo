# AIOpsLab: A Holistic Framework to Evaluate AI Agents for Enabling Autonomous Clouds

Section-by-section notes

Paper: Chen, Shetty, Somashekar, Ma, Simmhan, Mace, Bansal, Wang, Rajmohan (UIUC, UC Berkeley, Microsoft, IISc). arXiv:2501.06706v1, January 2025.

---

# Abstract

**What problem they look at**
- AIOps means using AI to do IT operations work, like finding where a fault is (localization) and why it happened (root cause analysis).
- Most older tools and AIOps algorithms solve **one task only**. For example, one model does anomaly detection, and a different one does localization.
- LLM agents can now do **many tasks in a row**, from start to end.

**Their big idea: AgentOps**
- They imagine AI agents that handle the **full incident lifecycle** by themselves. The end goal is a "self-healing" cloud.
- They call this idea **AgentOps** (Agents for Operations).

**What they built: AIOpsLab**
- To build and test such agents, you need a proper framework. AIOpsLab does these things:
  - Deploys a microservice application on a cloud setup
  - Injects faults into it
  - Generates user traffic (workload)
  - Collects and exports telemetry (logs, metrics, traces)
  - Connects all these parts together
  - Gives the agent an interface to work with the system, and then scores the agent

**What they test**
- They run current LLM agents on a benchmark made with AIOpsLab and report what the agents do well and where they fail.

---

# Section 1: Introduction

## Part 1: Why this matters
- Companies now run on large cloud systems built with microservices and serverless.
- These designs scale well, but they are **complex**. One small problem can spread and cause a big outage.
- Example from the paper: an Amazon outage can lose about **$100 million in one hour** (this is from Prime Day 2018).

## Part 2: The goal of AIOps
- AIOps sits inside DevOps. Its final goal is a cloud that can **detect, localize, and fix** faults with very little human help.
- This idea is not new. Papers from 2009 and 2012 already talked about self-healing clouds.
- What is new: LLM agents. An agent can use **external tools** (run commands, read logs, call APIs) and react to what it sees. So it can, in theory, handle the whole lifecycle.

**Figure 1** shows this lifecycle:
- On the left: a small microservice system. A frontend service talks to a storage service, and there is a slow connection between them.
- On the right: the 4 stages of incident handling:
  1. Incident detection
  2. Triage after localization (find which part is broken)
  3. Root cause diagnosis
  4. Mitigation / resolution
- Either human operators or agents can do these stages.

## Part 3: AgentOps in more detail
- In AgentOps, an agent does not stop at one task. It works on **multiple tasks across different layers** of the system (application, Kubernetes, network, etc.).
- The agent makes decisions in real time and takes actions from start to end.
- They support this with a quote from a Berkeley blog post (Zaharia et al., 2024). The main point: the best AI results now come from **systems made of many parts** (models + tools + logic), not from a single model alone.

## Part 4: The gap they want to fill
- The **"Dev"** side of DevOps already has good benchmarks for agents: SWE-bench, HumanEval, LiveCodeBench, WebArena, R2E.
- The **"Ops"** side does not. There are no good benchmarks with **realistic and varied** incident scenarios.
- So the main contribution of this paper is a framework to design, build, and evaluate AIOps agents in an **interactive** environment.

## Part 5: Challenges

**Challenge 1: Putting all the pieces together**
- The evaluation flow must:
  - Work for many different agents and many different cloud apps
  - Be strong enough to test complex, realistic tasks
  - Give useful feedback and observability to the agent
  - Be easy to extend with new tasks and new agents
- Tools already exist for each piece, but **separately**:
  - Observability tools (for example, trace sampling, syscall-level monitoring)
  - Benchmark apps (DeathStarBench, TrainTicket, μSuite)
  - Chaos engineering tools (Netflix Chaos Monkey, ChaosBlade, ChaosMesh)
- No one has joined them into **one** system for AIOps evaluation.
- Their answer: AIOpsLab handles the full flow (deploy, inject fault, generate workload, run the agent, analyze results). The key part is the **Agent-Cloud Interface (ACI)**. This is how the agent talks to the cloud, takes actions, and gets feedback.

**Challenge 2: No realistic test scenarios**
- Most AIOps benchmarks use **static data**:
  - Saved metric time series (for example ADBench, AnomalyBench)
  - Fixed question-and-answer sets (for example OpsEval)
- Static data does not behave like a real system. In a real cloud, workloads and incidents change over time, and an agent's actions change the system.
- Recent agent papers for operations (like RCAgent) use **private company services and data**, so others cannot repeat the results.
- Most benchmarks also test only **one stage**, like only anomaly detection or only localization. So you cannot compare agents across the full lifecycle, or decide which agent is best for a given situation.

**Their answer to Challenge 2**
- They create **"problems"**. Each problem is a realistic incident inside a running microservice system.
- Problems are grouped by a **task taxonomy** (detection, localization, root cause analysis, mitigation). Details come in Section 2.
- They go beyond simple faults like "the pod crashed" or "it got slow". Those faults have no deeper cause to find or fix. They add faults with **real root causes**, like misconfigurations, so the agent's diagnosis and repair skills are really tested.

## Part 6: How they built it (short overview)
- **Test apps:** two apps from DeathStarBench (SocialNetwork and HotelReservation), with their workload generators.
- **Faults:** their own fault library, plus ChaosMesh.
- **Telemetry:**
  - Prometheus for metrics
  - Jaeger for traces
  - Filebeat + Logstash for logs
  - All data is also saved to disk. So you can test **old non-LLM AIOps methods** on the same data, not only agents.
- **Control:** Helm and the Kubernetes API, used by the orchestrator.
- **Benchmark size:** 48 problems, 4 agents tested.
- Adding a new agent takes **less than 100 lines of code**.

## Part 7: List of contributions
1. They explain what a full framework for AIOps agents needs, and what makes it hard.
2. They build AIOpsLab (deploy, inject, workload, telemetry, orchestration, agent interface).
3. They use it to make a benchmark of 48 problems and test 4 LLM agents.
4. They analyze where the agents do well and where they fail.
5. They promise to release it publicly. (The paper says "link will be provided". It was later released on GitHub by Microsoft.)

## What this means for you
- This paper sits very close to your incident diagnosis work. The two critiques in Challenge 2 are worth noting: **static datasets** and **testing only one stage**. Reviewers may ask the same questions about any AIOps method that is tested only on saved data.
- AIOpsLab could work as a **live testbed** for your own pipeline, since it can also export telemetry for non-LLM methods.

---

# Section 2: AIOpsLab Design

It has 5 parts: 2.1 Problem Definition, 2.2 Orchestrator, 2.3 Cloud Services, 2.4 Fault Library, and 2.5 Observability.

## 2.1 Problem Definition

### The basic formula
Every test case is called a **problem**. They write it as:

**P = ⟨T, C, S⟩**

- **T = Task.** What the agent must do. There are 4 types:
  - Detection
  - Localization
  - Root cause analysis (RCA)
  - Mitigation
- **C = Context.** The situation the problem happens in, plus the information given to the agent.
- **S = Solution.** The correct answer (also called the "oracle"). It is used to score the agent.

Each task type has its own success rule and metrics. Example: detection uses **Time-to-Detect (TTD)**, which is how long it took to notice the fault.

### The context C is split into two parts
**C = ⟨E, I⟩**

- **E = Operational environment.** This is **hidden** from the agent. It includes:
  - The cloud service (the app)
  - The fault model (what fault was injected)
  - The workload model (what traffic was sent)
- **I = Problem information.** This is **given** to the agent. Two kinds:
  - **Direct:** service description, task description, and documentation of the APIs the agent can use
  - **Indirect:** logs, metrics, and traces. These are not handed over at the start. The agent has to **ask for them** while it works.

So the agent knows what the app is and what tools it has, but it does not know what fault was injected. It must find out from telemetry.

### How the solution S is checked
- S is designed separately for each problem and task.
- Some problems, especially **mitigation**, can be fixed in more than one way.
- For these, AIOpsLab does not only check the one service where the fault was injected. It checks the **whole system**, for example "are all services up and running?"
- Reason: while fixing one thing, the agent may break something else by accident. Checking the full system catches this.

### Example 2.1 (code)
The problem: **localize a Kubernetes target port misconfiguration** in the SocialNetwork app.

What the code does, line by line:
- It extends a ready-made class called `LocalizationTask`.
- `self.app = SocialNetwork()` sets the app.
- `self.ans = "user-service"` sets the correct answer. This is also the service where the fault will be injected.
- `start_workload()` uses the `wrk` tool to send 100 requests per second for 10 seconds to the app's frontend.
- `inject_fault()` uses `VirtFaultInjector` (virtualization-level injector) to inject `"misconfig_k8s"` into user-service.
- `eval()` records:
  - `TTL` (time to localize) = how long the agent took
  - `success` = whether the agent's answer is an **exact match** with "user-service"

Mapping back to the formula:
- T = localization
- S = "user-service"
- C = SocialNetwork app + misconfiguration fault + wrk workload

Two small things to notice:
- Success is an **exact string match**. So the agent must give the exact service name. A close or partly correct answer counts as wrong.
- In the printed code, the variable `res` is used without being created first. It is just a short version of the code for the paper, not the full code.

## 2.2 Orchestrator

The Orchestrator is the **central controller**. It keeps the agent and the cloud service separate. The agent never touches the cloud directly. Everything goes through the Orchestrator.

### Figure 2 (overview of the whole system)
Colors in the figure:
- **Green** = provided by AIOpsLab (fault generator, workload generator, telemetry collector, common evaluation, task types)
- **Blue** = can be extended by the developer (agents, fault library, workload policy, apps under test, customized evaluation, problem definitions)
- **Pink** = evaluation result

The numbered steps:
1. **Register:** the agent registers with the Orchestrator.
2. The Orchestrator **deploys the app** (SocialNetwork, HotelReservation, or others).
3. The Orchestrator calls the **fault generator**.
4. The fault generator, using the fault library, **injects the fault** into the app.
5. The Orchestrator calls the **workload generator**.
6. The workload generator, following a workload policy, **sends traffic** to the app.
7. The agent sends an **action** (for example, a command) to the Orchestrator.
8. The app sends its **state and telemetry** back to the Orchestrator, which passes it to the agent.
   (Steps 7 and 8 repeat many times.)
9. The agent **submits** its final answer S.
10. The Evaluator scores it and gives the **evaluation result**.

Inside the Orchestrator box you also see:
- **Telemetry Collector**
- **Evaluator** (common evaluation + customized evaluation)
- **Problem Pool**, made of problem definitions (task T, workload W, fault F, expected solution S') and task types (Detect, Localize, RCA, Mitigate)

### 2.2.1 Agent-Cloud Interface (ACI)

**Why it is needed**
- Humans use the cloud through APIs, CLIs, dashboards, and incident portals.
- These were made for humans, not for LLMs.
- A human can easily ignore unimportant information. An LLM gets **distracted** by it, and this hurts its performance.

**What the ACI defines**
1. The **list of actions** the agent is allowed to take (steps 7 and 9 in Figure 2)
2. **How the result** of each action is sent back to the agent as an observation (step 8)

**Default APIs**
- `get_logs` to fetch logs
- `get_metrics` to fetch metrics
- `get_traces` to fetch traces
- `exec_shell` to run shell commands, but only after passing **security policy filters** (so the agent cannot run anything it wants)

**Example 2.2 (code for `get_traces`)**
- Inputs: the Kubernetes namespace, and a duration (default 5).
- It takes the current time as the end time, and goes back by the given duration to get the start time.
- It pulls traces from **Jaeger** for that time window.
- It **saves the traces to a directory** and returns the **path** to that directory, not the trace data itself.

Remember this last point. In Section 3 they find that agents often open these files with `cat` and flood their own context with too much data.

**Other details**
- When a problem starts, the Orchestrator **automatically reads the documentation** (the docstrings) of these APIs and gives it to the agent as part of context C.
- Using the Orchestrator's privileged access, the agent can also do actions like **scaling, redeploying, and patching** services.
- After each action, the agent gets feedback: the **output, error messages, and tracebacks**.

### 2.2.2 Session Interface

- The Orchestrator also manages the **lifecycle** of both the agent and the app.
- Every time one agent works on one problem, a new **Session** is created.
- A session starts when the agent registers and passes a **unique problem ID** (step 1).
- To connect any agent, there is **only one requirement**. The agent must have this method:

  ```
  async def get_action(state: str) -> str
  ```
  - Input: the current state of the service (sent by the Orchestrator)
  - Output: the next action the agent wants to take
- This can be a small wrapper around any existing agent framework. So it works with many agent tools, not just one.

**Example 2.3 (code)**
- An `Agent` class builds a prompt from the problem description, the instructions, and the APIs. It uses GPT-4 as the LLM.
- `get_action` just sends the prompt plus the current state to the LLM and returns the reply.
- Then:
  - Create the Orchestrator.
  - Pick the problem ID `"misconfig_app_hotel_res-mitigation-1"` (a misconfiguration in HotelReservation, mitigation task).
  - `init_problem(pid)` returns the problem description, instructions, and APIs.
  - Register the agent with the name "myAgent".
  - Start the problem with **max_steps = 10**. So the agent gets at most 10 actions.
- After this, the Orchestrator keeps **asking the agent for its next action** (by calling `get_action`) until the agent submits or runs out of steps.

### 2.2.3 Other Interfaces

**Problem Initializers** (setting up the problem)
- The Orchestrator deploys the app using infrastructure-as-code tools like **Helm**.
- Then it uses two generators to create the incident:
  - **Workload generator** (step 5). Right now this is **wrk2**. It supports different workload policies and can also **replay real industry workloads** (step 6). Other generators can be added.
  - **Fault generator** (step 3). It uses their own fault library, which can inject faults at different levels of the system, such as the **application level** and the **virtualization level** (step 4).
- The faults are **fine-grained and have parameters**. They are meant to need real diagnosis and fixing, not just surface symptoms.

**Problem Evaluators** (scoring the agent)
- The Orchestrator compares the agent's answer with the success rules of that task (step 10).
- Default metrics include:
  - Time-to-Detect for detection
  - Number of steps taken
  - Number of tokens used by the LLM agent
- Optional: **LLM-as-a-Judge**. Another LLM reads the agent's full trajectory and gives a qualitative judgment.
- Any extra metrics the user defined for that problem are also run. Example: in Example 2.1, success = exact name match.
- The Orchestrator **logs every trajectory**: every action and every resulting system state. This helps later analysis and debugging.
- All results are collected automatically.

## 2.3 Cloud Services

- AIOpsLab deploys **live** microservice apps (step 2 in Figure 2).
- Right now it supports two apps from **DeathStarBench**:
  - **SocialNetwork:** 28 microservices, including Memcached, MongoDB, and Redis. It copies features of a real social network.
  - **HotelReservation:** written in Go with gRPC. It has services for recommending and reserving hotels.

## 2.4 Task-oriented Fault Library

### 2.4.1 Task Taxonomy (Table 1)

The 4 tasks are ordered by **level**. A higher level is **harder** and has **more impact**.

| Level | Task (sub-tasks) | What it tests |
|---|---|---|
| 1 | Detection (1) | Can it notice that something is abnormal? |
| 2 | Localization (1) | Can it find the exact source, for example which microservice? |
| 3 | RCA (2) | Can it find the real cause of the fault? |
| 4 | Mitigation (1) | Can it fix the system? |

- Level 1 example: detecting that a Kubernetes pod of a microservice is not working correctly.
- RCA has **2 sub-tasks**:
  1. Which **system layer** is affected (for example, application or virtualization)
  2. What **type of fault** it is (for example, misconfiguration or operation error)
- Users can add harder tasks or their own sub-tasks.
- To create problems for these levels, they inject faults and build a **problem pool**.

### Figure 3: two kinds of faults
- **Symptomatic faults.** Examples: network loss, pod failure.
- **Functional faults.** Split into application-level faults and virtualization-level faults.

### 2.4.2 Symptomatic Faults
- These are faults like **slowdowns** and **crashes**.
- You can see them as symptoms: higher latency, resources running out, or services going down.
- They are used for **Level 1 and Level 2** (detection and localization).
- They **do not have a deeper root cause** to find. The fault is the symptom itself. So they cannot be used for RCA or mitigation tasks.
- AIOpsLab uses **ChaosMesh** to inject them.

### 2.4.3 Functional Faults

**The problem with existing fault injection tools**
- There are many fault injection tools for cloud systems (they cite more than 20).
- Most of them only inject **symptoms**. These are "coarse-grained" faults. They break the system, but they do not create a real underlying cause like a misconfiguration or a software bug.
- So they cannot test whether an agent can **diagnose and fix** a root cause.

**What functional faults add**
- They copy realistic causes such as **wrong deployment or wrong operations**.
- The agent must do all 4 levels: detect (L1), localize (L2), find the root cause (L3), and apply the right fix (L4).

**Figure 4: Revoke authentication fault**
- It shows part of the HotelReservation app:
  - Frontend calls Search, User, and Recommend.
  - Search calls Rate and Geo.
  - Geo uses its database, **Mongodb-geo**.
- The fault: while the app is running, the **admin privilege on Mongodb-geo is removed**.
- What you see: the **Geo service** starts showing errors like "Not authorized on geo-db to execute command."
- Important point: the fault is injected in **Mongodb-geo**, but the error messages appear in **Geo**. The place where you see the symptom is not the place where the fault is. An agent that only reads error logs may blame the wrong service.

**Example 2.4 (code)**
- An `ApplicationFaultInjector` class with two methods:
  - `inject_revoke_auth(microservices)` removes MongoDB admin privileges
  - `recover_revoke_auth(microservices)` gives them back
- Usage: a problem class `MongoDBRevokeAuth` sets the app to HotelReservation, and injects `"revoke_auth"` into `"mongodb-geo"`.

**What users can do with the library**
- Choose different target services for the same fault
- Inject **multiple faults into multiple services at the same time**
- Write their own custom faults
- For each fault, AIOpsLab gives both an **inject** function and a **recover** function. So the system can always be brought back to a healthy state after the test.
- The full problem pool is described in Section 3.3.

## 2.5 Observability

AIOpsLab has a telemetry layer that can be extended. It collects 3 main kinds of data:
1. **Traces** from **Jaeger**. These show the full path of each request through the services.
2. **Application logs**, fetched with **kubectl**, or collected and formatted with **Filebeat and Logstash**.
3. **System metrics** from **Prometheus**.

Other points:
- Data is collected **while the agent works**, and it can also be **exported offline**. Offline data lets you test traditional (non-LLM) AIOps methods on the same incidents.
- It can also capture other information: **codebase, configuration, and cluster information**.
- Developers can expose **low-level system data**, for example **syscall logs**, to the agent through the AIOpsLab interface.

## What this means for you
- The last point in 2.5 matters for you. Syscall and kernel-level data are **not built in**. The paper only says you *can* add them. Telemetry is limited to logs, metrics, and traces. Adding kernel trace data as a new API could be a real contribution on top of AIOpsLab.
- Figure 4 shows a pattern to keep in mind for any diagnosis pipeline: the service that shows errors (Geo) is not the one with the fault (Mongodb-geo). An evaluation that uses exact name match will mark "Geo" as wrong.
- `get_traces` returns a file path, not a summary. How an agent reads and filters this data is left to the agent. Section 3 shows this becomes a real problem.

---

# Section 3: Evaluation

It has 6 parts: 3.1 Setup, 3.2 Metrics, 3.3 Problem Pool, 3.4 Results, 3.5 Step Limit, and 3.6 Agent Behavior.

The opening paragraph just lists the plan: setup and metrics, then the faults in Table 2, then results, cost, and reasons for failure. All results are collected automatically by the evaluators from Section 2.2.3.

## 3.1 Evaluation Setup

**LLM-based agents they test**
- All agents are used **as they are**, with no fine-tuning or changes. This keeps the comparison fair.
- **GPT-3.5-Turbo with shell** and **GPT-4-Turbo with shell** (written as GPT-w-Shell). These are the simple baselines. The LLM only has access to a secure shell.
- **ReAct.** An agent that switches between **thinking** (writing its reasoning) and **acting** (calling a tool). It builds on chain-of-thought.
- **FLASH.** An agent made for cloud operations. It:
  - Watches the execution status
  - Breaks a complex instruction into smaller steps with conditions
  - Uses **hindsight**: it learns from what happened in past steps

  FLASH was **not public** when they wrote this paper. So they built a **simplified version** themselves. Their version writes down insights after each step.

**Non-LLM methods they test (traditional AIOps)**
- These use telemetry data as input and each one does only one task:
  - **MKSMC** for detection
  - **RMLAD** and **PDiagnose** for localization

One thing to keep in mind: the FLASH paper comes from the same Microsoft group as this paper, and the FLASH tested here is their own simplified rebuild, not the original.

## 3.2 Metrics

**Correctness**
- Did the agent detect, localize, analyze, or fix the problem correctly?

**Time and Steps**
- **Time-to-Detect (TTD):** time from when the fault happens to when it is detected.
- **Time-to-Mitigate (TTM):** time from detection until the fault is fully fixed.
- **Number of steps:** how many times the agent **interacted with AIOpsLab**. This is not the number of calls to the LLM. One step could include several LLM calls inside the agent.

**Cost**
- Measured as **number of tokens** (input + output).

## 3.3 Problem Pool

### Size
- **48 problems** in total.
- The text says "with six agents, we evaluate 288 cases" (48 × 6 = 288). But Section 3.1 lists 4 LLM agents and 3 non-LLM methods. The paper does not explain the "six". It is a small inconsistency.

### Table 2: The 10 faults used

The "Ext." column says how easy it is to reuse the fault to make new problems: easy, some manual work, or only usable for specific problems.

| No. | Fault | App | Levels | Category | Reuse | # Problems | What happens |
|---|---|---|---|---|---|---|---|
| 1 | AuthenticationMissing | Hotel | 1-4 | Functional, Virtualization | Some manual work | 4 | Missing login credentials, MongoDB denies access |
| 2 | TargetPortMisconfig | Social | 1-4 | Functional, Virtualization | Easy | 12 | Service cannot connect because the port is set wrong |
| 3 | RevokeAuth | Hotel | 1-4 | Functional, Application | Some manual work | 8 | Revoked permissions break database connection |
| 4 | UserUnregistered | Hotel | 1-4 | Functional, Application | Some manual work | 8 | Database access fails after the user is removed |
| 5 | BuggyAppImage | Hotel | 1-4 | Functional, Application | Specific only | 4 | A bug in the connection code inside the app image |
| 6 | ScalePod | Social | 1-4 | Functional, Virtualization | Easy | 4 | Wrong scaling sets a service's pod count to zero |
| 7 | AssignNonExistentNode | Social | 1-4 | Functional, Virtualization | Easy | 4 | Pod stuck in Pending because it is assigned to a node that does not exist |
| 8 | NetworkLoss | Hotel | 1-2 | Symptomatic | Easy | 2 | Network loss breaks communication for one service |
| 9 | PodFailure | Hotel | 1-2 | Symptomatic | Easy | 2 | A pod fails and the service stops |
| 10 | Noop | Both | 1 | None | Easy | 2 | **No fault injected.** Normal system. |

Main points:
- **Functional faults (1 to 7)** create problems at **all 4 levels**.
- **Symptomatic faults (8, 9)** only create **detection and localization** problems, because they have no deeper cause to find or fix.
- **Noop (10)** is a "nothing is wrong" test. It checks if the agent wrongly reports a fault.

Note: the problem counts in Table 2 add up to **50**, not 48. The 2 Noop problems seem to be the difference (see 3.6.4 below).

### What each task asks the agent to give
- **Detection (Level 1):** just "yes" (there is a fault) or "no". A binary answer. They say it could be made harder, like asking the agent to mark which telemetry is abnormal, but they kept it simple.
- **Localization (Level 2):** the exact place of the fault, usually a **service name or pod name** in Kubernetes.
- **RCA (Level 3):** two answers:
  1. Which **system layer** is affected
  2. What **type of fault** it is (for example, misconfiguration or operation error)
- **Mitigation (Level 4):** the agent must actually **fix** the system with a series of actions, like updating a config or rolling back to an older version.

### Reusing faults to make new problems
- **Easy example, Fault 2:** it can be injected into **10 different services** just by changing the target. They picked 3: `user-service`, `text-service`, and `post-storage-service`.
- **Why different targets matter:** each service has different dependencies. So the same fault has a different **"blast radius"**, meaning it spreads through the system in a different way. Different targets give different difficulty.
- **Harder examples:**
  - Faults 3 and 4 need scripts to remove the permission or the user during the test, **and** an update to the app's Kubernetes ConfigMap.
  - Fault 1 needs a Helm config update to enforce TLS.
- **Not reusable:** Fault 5 is a code bug built into one microservice's image. It only works for that specific problem.

## 3.4 Performance Results

### Table 3: Overall results

| Agent | Lines of code to register | Avg time (s) | Avg steps | Avg tokens | Accuracy |
|---|---|---|---|---|---|
| GPT-4-w-Shell | 41 | 28.61 | 6.44 | 6,394.5 | 49.15% |
| GPT-3.5-w-Shell | 41 | 12.44 | 14.70 | 2,557.95 | 15.25% |
| ReAct | 49 | 43.79 | 11.50 | 16,941.46 | 55.93% |
| FLASH | 60 | 99.64 | 8.48 | 6,484.25 | **59.32%** |

What it shows:
- **FLASH** has the best accuracy, but it is the **slowest** (almost 100 seconds per problem).
- **GPT-3.5** is the fastest, but has the **lowest accuracy** (15.25%). It also takes the most steps, which means it wastes steps (explained in 3.6).
- **ReAct** uses the **most tokens** by far (about 17k), almost 3 times GPT-4 and FLASH.
- All agents need **less than 100 lines of code** to connect to AIOpsLab (41 to 60).

### Table 4: Results by task

**(a) Detection**

| Agent | Accuracy | Time (s) | Steps | Input tokens | Output tokens |
|---|---|---|---|---|---|
| GPT-4-w-Shell | 69.23% | 7.08 | 3.85 | 5,492 | 132 |
| GPT-3.5-w-Shell | 23.07% | 11.05 | 13.60 | 1,940 | 386 |
| ReAct | 76.92% | 39.00 | 11.46 | 15,608 | 933 |
| FLASH | **100%** | 78.27 | 6.77 | 12,869 | 126 |
| MKSMC (non-LLM) | 15.38% | 1.00 | N/A | N/A | N/A |

- Detection is just yes or no, so it should be the **easiest** task. But only **FLASH** gets all of them right.

**(b) Localization**

| Agent | Acc@3 | Acc@1 | Time (s) | Steps |
|---|---|---|---|---|
| GPT-4-w-Shell | 61.54% | **61.54%** | 7.04 | 4.23 |
| GPT-3.5-w-Shell | 30.77% | 30.77% | 6.26 | 11.92 |
| ReAct | **69.23%** | 53.85% ↓ | 38.65 | 11.08 |
| FLASH | 61.54% | 46.15% ↓ | 56.60 | 5.77 |
| PDiagnose (non-LLM) | 15.38% | 15.38% | 1.02 | N/A |
| RMLAD (non-LLM) | 7.69% | 7.69% | 1.98 | N/A |

- Agents can give a **list** of suspect services, since more than one fault could happen at the same time.
- **Acc@1:** is the first answer correct? **Acc@3:** is the correct answer in the top 3?
- **ReAct** is best at top 3 but drops at top 1. FLASH drops too.
- **GPT-4** is the same in both. When it is right, its first guess is right. It is the **best at Acc@1**, and also the fastest.

**(c) RCA**

| Agent | Accuracy | Time (s) | Steps |
|---|---|---|---|
| GPT-4-w-Shell | 40.90% | 8.68 | 4.81 |
| GPT-3.5-w-Shell | 9.09% | 10.06 | 14.00 |
| ReAct | **45.45%** | 32.16 | 8.00 |
| FLASH | 36.36% | 59.00 | 6.09 |

**(d) Mitigation**

| Agent | Accuracy | Time (s) | Steps | Input tokens |
|---|---|---|---|---|
| GPT-4-w-Shell | 27.27% | 99.47 | 13.72 | 10,143 |
| GPT-3.5-w-Shell | **0%** | 23.78 | 20.00 | 3,178 |
| ReAct | 36.36% | 67.18 | 15.54 | 29,212 |
| FLASH | **54.55%** | 216.41 | 16.09 | 8,469 |

- RCA and mitigation are the **hardest** tasks.
- **GPT-3.5 fixed nothing.** Its average of exactly 20 steps means it always used up all its steps.
- FLASH is best at mitigation, but takes over **3.5 minutes** per problem.
- ReAct sends almost **30k input tokens** per mitigation problem.

**Their main conclusion here:** no agent is good at **all 4 tasks**. Even the best one (FLASH) struggles with harder tasks like mitigation.

### My check on the numbers
Working backwards from the percentages to find how many problems are in each task:
- Detection: 13 problems (9/13 = 69.23%)
- Localization: 13 problems
- RCA: 22 answers. That fits 11 problems × 2 sub-tasks (layer + fault type).
- Mitigation: 11 problems (6/11 = 54.55%)

13 + 13 + 11 + 11 = **48 problems**. This matches the paper.

The overall accuracy in Table 3 fits this too. For example, FLASH: 13 + 8 + 8 + 6 = 35 correct out of 59 answers = 59.32%. This also tells you Table 3 uses **Acc@3** for localization, not Acc@1.

Why this matters:
- The sample size is **small**. In detection and localization, **one problem = 7.7%** accuracy. In mitigation, **one problem = 9.1%**.
- So a difference like ReAct 36.36% vs GPT-4 27.27% in mitigation is just **one problem**. The paper does not report repeated runs or variance.

## 3.5 Influence of the Step Limit

### Figure 5: Accuracy vs. max number of steps (K)
The x-axis is the step limit: 3, 5, 10, 15, 20. Rough reading of the plot:

| K | GPT-4 | GPT-3.5 | ReAct | FLASH |
|---|---|---|---|---|
| 3 | ~0.15 | ~0.08 | ~0 | ~0 |
| 5 | ~0.42 | ~0.15 | ~0.07 | ~0.30 |
| 10 | ~0.46 | ~0.15 | ~0.32 | ~0.52 |
| 15 | ~0.46 | ~0.15 | ~0.51 | ~0.56 |
| 20 | ~0.49 | ~0.15 | ~0.56 | ~0.59 |

What it shows:
- **ReAct and FLASH** need many steps. They improve a lot as you give them more steps. FLASH hits 59.32% at 20 steps.
- **GPT-4** gets most of its accuracy in the first 5 steps, then only grows a little.
- **GPT-3.5** stops improving after 5 steps. More steps just means **more tokens wasted**.

### Their explanation
- Accuracy **flattens** after some number of steps. They say this means **"self-repair from environment feedback"** reaches its limit quickly in AIOps. In simple words: just trying again and looking at the result stops helping after a while.
- Compare with **Dev tasks** like coding. There, agents get useful feedback from **linters, type checkers, and test cases**, and they keep improving step by step. In Ops, there is no such clear signal saying "you are closer now".

One thing to notice: in the figure, ReAct is still going up at K = 20. So the "flattening" is clear for GPT-3.5 and GPT-4, but less clear for ReAct and FLASH.

### What they say is needed
1. Better **task breakdown using planning**
2. Better **feedback for the steps in the middle**, not only at the end
3. Methods that go **beyond** just environment feedback and self-repair

## 3.6 Agent Behavior: what went well, what went wrong

### Overall
- **All LLM agents beat the non-LLM methods** on detection and localization. Compare Table 4: MKSMC 15.38%, PDiagnose 15.38%, RMLAD 7.69%.

### Figure 6: Which actions agents used (ReAct and FLASH only)

| Action | ReAct | FLASH |
|---|---|---|
| get_logs | 25.5% | 35.1% |
| get_metrics | 5.8% | 1.3% |
| get_traces | 4.1% | **0.0%** |
| Others | 16.4% | 5.4% |
| K8S commands | 48.2% | 58.1% |

- **Kubernetes commands** (like `kubectl`) are the most used actions overall.
- Among the **telemetry APIs**, `get_logs` is used most, then `get_metrics`, then `get_traces`.
- **FLASH never uses traces at all.** And it still has the best overall accuracy.

### Table 5: Other shell commands used

| Agent | find | echo | py | awk | mongo | grep | ls | cat | ip |
|---|---|---|---|---|---|---|---|---|---|
| ReAct | 0 | 0 | 0 | 3 | 0 | 1 | 26 | **30** | 0 |
| FLASH | 0 | 3 | 0 | 0 | 0 | 0 | 8 | 10 | 0 |

- Mostly `ls` and `cat`. They list files and print them out.
- Almost no filtering tools (`grep` only once, `awk` only 3 times). So agents read raw files instead of searching inside them.

### 3.6.1 Wasting steps on useless actions
Agents waste steps by:
- Calling the **same API again and again**
- Calling **APIs that do not exist**
- Spending too many steps **talking between agents** (in multi-agent setups)

Details:
- **GPT-3.5** often makes wrong API calls **in a loop** and gets the same error each time.
- In a multi-agent setup with `speaker_selection_method = round_robin`, every agent speaks every step. This stops agents from making a clear decision. They keep calling telemetry APIs to get "more information".
- With `speaker_selection_method = auto` (the system picks who speaks next), the chosen agent **talks 10 times in one step** without talking to others (10 is the max rounds per step).

The paper does not clearly say which tested agent used this multi-agent setup. These setting names look like a multi-agent library such as AutoGen, but the paper does not name it.

### 3.6.2 Too much data when reading telemetry

**Figure 7: Actions in successful vs failed cases**

| Action | Successful cases | Failed cases |
|---|---|---|
| Submit | 23.5% | 20.5% |
| kubectl get | 23.0% | 22.3% |
| kubectl other | 25.6% | 16.9% |
| get_logs | 16.7% | 17.4% |
| get_metrics | 2.6% | **8.2%** |
| get_traces | 2.6% | **5.8%** |
| Others | 6.0% | 8.9% |

What it shows:
- In **successful** cases, agents use metrics and traces **rarely** (5.2% together), only when needed.
- In **failed** cases, they use them **much more** (14.0% together).
- Successful cases also use more `kubectl` commands that are not just `get` (25.6% vs 16.9%). These are likely actions that change or inspect the system more deeply.

Their explanation:
- **Metrics** (like CPU and memory) have lots of numbers. These are hard for an LLM to understand as raw text.
- **Traces** describe how services depend on each other. They are easier to understand as a **picture**, not as raw text.
- Agents often open this data with `cat`. This **fills the context window**, distracts the model, and uses many tokens.
- So using telemetry without care can add **noise** to the agent's reasoning, and can even use up all its tokens.
- They expect future agents to need **better processing and filtering** of telemetry.

A point to think about: this is a **correlation**. It could also be that harder problems make the agent look at metrics and traces more, and those harder problems fail anyway. The paper does not test which one is the real reason.

### 3.6.3 Wrong API usage
- Agents often **format API calls wrongly**, for example with bad parameters.
- **GPT-3.5** uses the right API name but the wrong format, and **repeats the same mistake**. It often says "I apologize for the error. Here is the API call again:" and then makes the same error. This happened up to **14 times in a 20-step run**.
- **ReAct** also makes mistakes, but it usually **fixes itself** by reasoning. Their example:
  - ReAct calls `get_logs("test-social-network", "Social Network")`. "Social Network" is not a real service name.
  - AIOpsLab replies: "Error: Your service/namespace does not exist."
  - ReAct thinks: "I should check which services are deployed in this namespace."
  - ReAct runs `kubectl get services -n test-social-network` to find the correct name.

So the difference between a weak and a strong agent is not "never makes mistakes". It is "can it **recover** from mistakes".

### 3.6.4 False positives in detection
- They made 2 **Noop** problems (one per app) with **no fault**.
- **Only GPT-4-w-Shell** correctly said "no fault" in both.
- All others reported a fault. They mistook **normal things**, like the normal workload traffic, as a problem.

This also explains the counting issue from earlier:
- FLASH has **100%** detection in Table 4, but it **fails** the Noop problems.
- So the Noop problems must **not be included** in Table 4. This is why Table 2 adds to 50 but the benchmark is 48.
- This means all 13 detection problems in Table 4 have a fault, and the right answer is **always "yes"**. An agent that always says "yes" gets 100%. So FLASH's 100% partly shows it tends to say "yes", not only that it detects well. The Noop result confirms this.

## What this means for you
- **Small sample sizes.** One problem can change accuracy by 7 to 9%. If you compare your method to these numbers, report counts too (like 8/13), not only percentages. If you can, run each problem more than once.
- **Detection is one-sided.** Detection results without "no fault" cases can look better than they are. If you build your own test set, add normal cases so you can measure false positives.
- **Telemetry handling is a real gap.** The paper's own finding is that agents fail when they read raw metrics and traces. They suggest better **filtering and summarizing**, but they do not build it. This is close to your thesis area, so it could be a good direction.
- **Old models.** These results use GPT-3.5 and GPT-4-Turbo from early 2025. Newer models will likely score higher, so these numbers are a starting baseline, not a ceiling.

---

# Section 4: Discussion

It has 3 short paragraphs.

## Paragraph 1: Making your own scenarios
- AIOpsLab lets engineers **easily create their own incident scenarios** to test agents.
- The **ACI works as a guard-rail**. The agent can only use the allowed actions, so testing happens in a **controlled environment**.
- Because of this, users can spend their time on the important part:
  - Designing scenarios that look like **real incidents from their own systems**
  - Deciding what problems their agents should solve

## Paragraph 2: Adding other kinds of problems
- AIOpsLab can work with **other fault types** too.
- Example for detection: you could add an **anomaly detection workload scenario**.
- You could also make problems where the agent must **label** the workload or telemetry data to show **which parts are abnormal**. This is the harder detection task they mentioned in 3.3 but did not build.

## Paragraph 3: Checking the agent's reasoning, not just its answer
- For scoring, you may need **more detailed oracles** (answer checkers) or the optional **LLM-as-a-Judge**.
- **Why:** in the yes/no detection task, an agent can give the **right answer for the wrong reason**.
- **Their real example:** an agent said "yes, there is abnormal behavior". This was correct. But its explanation pointed to a workload that was **normal** and had **nothing to do** with the injected fault. So it was right by luck.
- **Their fix:** use LLM-as-a-Judge to compare the agent's **reasoning steps** with the real problem setup (the injected fault, the workload, and the environment).

How this connects to earlier sections:
- This supports what was noted in 3.6.4. Since all detection problems in Table 4 have a fault, an agent can get "yes" right without understanding anything.
- The LLM-as-a-Judge is **optional**, and the results in Section 3 **do not use it**. So the detection numbers in Table 4 only check the yes/no answer, not the reasoning.

---

# Section 5: Related Work

It has 2 short paragraphs.

## AgentOps (LLMs for cloud operations)
They list earlier work that uses LLMs for cloud operations:
- **Fine-tuned GPT models** (Ahmed et al., ICSE 2023). Recommend root causes and mitigation steps for cloud incidents.
- **RCACopilot** (Chen et al., EuroSys 2024). Automatic root cause analysis for cloud incidents.
- **RCAgent** (Wang et al., 2023). Root cause analysis using autonomous agents with tools.
- **MonitorAssistant** (Yu et al., FSE 2024). Helps with cloud service monitoring.
- **Xpert** (Jiang et al., ICSE 2024). Recommends queries for incident management.

These show that LLMs work well for monitoring and analyzing complex systems.

The problems they point out:
- Most of these have **no public code**
- Most use **private datasets**
- There is **no shared benchmark** with realistic scenarios to compare agents across **different operation tasks**

## AIOps benchmarks
- Existing AIOps benchmarks use **static or text-based data**:
  - Metric time series (ADBench, AnomalyBench)
  - Fixed question-and-answer sets (OpsEval)
- General LLM benchmarks have the same issue. They mention MMLU, HELM, BIG-bench, C-Eval, and others.
- None of these:
  - Simulate a **live, changing** cloud environment
  - Let agents **interact** with the system to solve tasks

## Things to notice about this section
- It is **very short**. Fault injection tools were covered back in 2.4.3, and coding benchmarks in the Introduction, but this section does not compare AIOpsLab against them in detail.
- Many of the cited LLM works (RCACopilot, MonitorAssistant, Xpert, the fine-tuned GPT work) come from the **same Microsoft group** as the authors.
- It does not compare with other **interactive** benchmarks for operations, if any existed at the time. For your own literature review, you would need to search wider than this list.

---

# Section 6: Conclusion

This repeats the main points. Nothing new is added:
- They explain what is needed, and what is hard, in building a framework to design, build, and test AIOps agents.
- They built **AIOpsLab**, which joins 4 parts:
  1. Fault injector
  2. Workload generator
  3. Cloud-agent orchestrator
  4. Telemetry observer
- It simulates cloud incidents and gives agents an **Agent-Cloud Interface** to work with the system and be scored.
- They built a benchmark of **48 problems** and tested **4 agents** across different AIOps tasks.

---

# Future directions spread across the paper

The paper has no separate Future Work section. Here is what it suggests, and where:

| Where | What they suggest |
|---|---|
| 2.5 | Give agents low-level data like **syscall logs** |
| 3.3 | Harder detection: agent **labels** the abnormal telemetry |
| 3.5 | Better **planning** to break tasks into steps |
| 3.5 | Better **feedback for middle steps** |
| 3.5 | Methods beyond "try again and look at the result" |
| 3.6.2 | Better **filtering and processing** of telemetry before the agent reads it |
| 4 | Check the agent's **reasoning** with LLM-as-a-Judge |
| 4 | New fault types and anomaly detection scenarios |

# Limitations noticed (the paper does not list these)
- Only **2 apps**, both from DeathStarBench.
- Faults are mostly **one at a time**. The framework supports many faults together, but they did not test that.
- **Small problem pool** (48), and each problem seems to be run **once**. No variance is reported.
- **Older models** (GPT-3.5 and GPT-4-Turbo).
- FLASH is a **simplified rebuild**, not the original.
- Localization uses **exact name match**, so a near answer counts as wrong.
- Detection results do **not include "no fault" cases**, so they can look better than they are.

# What this means for you
- If you write about AIOpsLab in your thesis, the clearest open gaps are: **telemetry filtering**, **low-level data like kernel traces**, and **checking reasoning, not only answers**. All three are things the paper suggests but does not build.
- If you use AIOpsLab to test your own pipeline, adding **multi-fault problems** and **more "no fault" cases** would make your results stronger than what this paper reports.
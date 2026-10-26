# OpenRCA: Can Large Language Models Locate the Root Cause of Software Failures?

Notes, section by section. Paper: Xu et al., ICLR 2025 (CUHK-Shenzhen, Microsoft, Tsinghua).

---

## Part 1: Abstract and Section 1 (Introduction)

### Abstract

**The problem they point at**
- LLM tools like Copilot and Cursor help with *writing* code.
- Most research is on these early stages of software work (code generation).
- The stage *after* software is deployed gets little attention: keeping it running and fixing failures. This stage matters a lot for users.

**What they built**
- **OpenRCA** is a benchmark (a dataset plus a way to score models). It tests if an LLM can find the **root cause** of a software failure.
- It has:
  - **335 failure cases**
  - from **3 enterprise software systems**
  - with **over 68 GB of telemetry** (logs, metrics, traces).
- The task: given a failure and its telemetry, the LLM must say what caused the failure.
- To do this, the model must understand how parts of the system depend on each other. It must also reason over very large, mixed data.

**Main result**
- Models do badly. They only solve the simplest cases.
- Even with their own agent (**RCA-agent**), the best model (Claude 3.5) solved only **11.34%** of cases.

### Section 1: Introduction

**Paragraph 1: Why this matters**
- LLM work in software engineering (MetaGPT, SWE-agent, OpenDevin, Copilot, Cursor) is mostly about the early part of the Software Development Life Cycle (SDLC).
- Running software in production and fixing problems is hard, slow work. It often needs people on call 24/7.
- Outages can cost billions. They give two real examples: the **CrowdStrike** outage and the **UniSuper / Google Cloud** incident.
- So better root cause analysis (RCA) is needed.

**Paragraph 2: What was tried before**
- Older AI methods for RCA:
  - **Causal discovery**: learning which signal causes which.
  - **Dependency graph analysis**: using the map of which service calls which.
  - **Other neural networks**.
- RCA is still hard for three reasons:
  - Real systems are very complex.
  - You need **multi-step reasoning**.
  - The data is huge and comes in different forms, and failures come in many patterns.
- Their main question: **Can current LLMs do RCA well?**

**Figure 1: The task in one picture**
- **Left:** a software system with **services** on top and **nodes** (machines) below. Red arrows show a failure **spreading** from one service to others. This is why RCA is hard. Many parts look broken, but only one is the real cause.
- **Middle:** the system produces telemetry (metrics, logs, traces), many GB of it.
- **Query (natural language):** "On March 25, 2021, between 09:00 and 09:30, there was a failure in system X. Find the root cause component and the root cause reason."
- **Output (JSON):**
  - `"component": "tomcat01"`
  - `"reason": "high disk I/O read usage"`
  - There can be a "root cause 2" too, because more than one failure can happen in the same window.
- The root cause has **3 parts**: **time, component, reason**. The query decides which of these the model must give.

**Paragraph 3: What OpenRCA is**
- 335 failure cases from 3 different real-world systems, with over 68 GB of **de-identified** telemetry (private details removed).
- Each case = a natural language query + telemetry. The model must output the asked root cause parts.
- The data comes in 3 shapes, and the model must reason across all of them:
  - **Time series**: metrics
  - **Dependency graphs**: traces
  - **Semi-structured text**: logs

**Paragraph 4: First results (no agent)**
- With **oracle telemetry**, Claude 3.5 solved only **5.37%**.
  - "Oracle" means engineers already picked the useful metrics for the model. This is a best case that is not realistic. It is explained in Section 4.1.
- With **balanced sampling**, it drops to **3.88%**.
  - Here the metrics are picked fairly but without knowing the answer. This is closer to real life.

**Paragraph 5: RCA-agent**
- It is a **multi-agent system** based on **writing and running code**. It builds on earlier work: TaskWeaver (Qiao et al.) and UFO (Zhang et al.).
- The key idea: the LLM does **not** read the raw telemetry. Instead, it writes **Python** to load and analyze the data, and only reads the results.
- What this gives:
  - The model's context is not filled with huge data.
  - The model can focus on reasoning.
  - It works even when telemetry is very large.
- With RCA-agent, Claude 3.5 goes up to **11.34%**.

**Paragraph 6: Closing**
- They hope OpenRCA becomes a standard benchmark for AI + software engineering in the operations stage.
- Code and data are public on GitHub.

### What this means for you
- Look at the 3 settings: oracle, balanced, and agent. "Let the LLM write code over the data" more than doubled the accuracy compared to "put sampled data into the prompt." If your TCH pipeline has an LLM step, this is a direct design point you can compare against.
- The low numbers are partly because scoring is strict. A case counts only if **every** asked part is correct. Section 2.3 explains this. Keep it in mind before you compare your own results with theirs.

---

## Part 2: Section 2 (OpenRCA)

Where the section points to the appendix, a short note is added.

### 2.1 Basics of root cause analysis

**What RCA means here**
- When a system fails (for example, a service goes down), on-call engineers collect telemetry and other info to find *why* it happened.

**A root cause has 3 parts**
- **Originating component**: which part failed first.
- **Start time**: when the failure began.
- **Failure reason**: why it failed. Examples: CPU overload, too much disk throughput.

**Why it is hard: failures spread**
- They spread in two ways:
  - **Through service calls.** Example: a payment service depends on a database. If the database fails, the payment service also looks broken.
  - **Through deployment.** Example: containers on the same server. If the server has a problem, all its containers suffer.
- So many components look broken at once. The job is to find the **first** one.

**The 3 types of telemetry**
- **Metrics**: time series of KPIs, like CPU usage or response time.
- **Traces**: records of how components call each other. They form a **graph** of dependencies.
- **Logs**: runtime messages from each component, with levels like info, warn, and error.
- Appendix A.3 shows real examples of each.

### 2.2 Features of OpenRCA

The authors list 5 things that make OpenRCA different:

1. **Real-world scenarios**
   - Older RCA datasets are either synthetic (made in a lab) or small.
   - OpenRCA has hundreds of real failures from 3 real systems.

2. **Goal-driven task design**
   - Older datasets usually test **one** goal only, such as "find the component."
   - Because of that, methods were built to fit one dataset and did not generalize.
   - OpenRCA writes the task as a **natural language question**. The question can ask for any mix of the 3 parts. This makes RCA one single task, and LLMs can take it as input easily.

3. **Large and mixed data**
   - Failures cover CPU, memory, and network problems at the container, node, and service levels (Table 5 in the appendix).
   - The model must reason over 3 data shapes: numbers (metrics), graphs (traces), and semi-structured text (logs).

4. **Full test of the LLM**
   - The model must understand the system's architecture, read real telemetry, and link clues from different data sources.
   - The authors say this goes beyond most software engineering tasks, like code generation.

5. **Can be updated**
   - New labels and data can be added later.
   - The reason is to keep it hard and avoid **data contamination**. This means models seeing the benchmark during training.

### 2.3 Task setup

**Input and output**
- Input: a natural language query plus the telemetry.
- Output: a subset of {time, component, reason} in **JSON**.
- The 3 parts can be combined into **7 tasks**:
  - 3 tasks ask for 1 part.
  - 3 tasks ask for 2 parts.
  - 1 task asks for all 3 parts.
- Appendix A.6 lists all 7 exactly (task_1 to task_7).

**How scoring works (important)**
- A case gets **1 point only if every part asked for is correct**. If even one part is wrong, it gets **0**.
- Accuracy = the average over all cases.
- To avoid problems with wording, the prompt gives the model the **full list of possible components and reasons**. So the model picks from a list. It does not write free text.
  - Example: "high disk I/O read usage" vs "disk read too high" would not count as a mismatch, because the model must pick the exact item from the list.
- Appendix A.7 adds one detail: if the model answers a part that was **not** asked for, it is ignored. It does not hurt the score.

### 2.4 How the dataset was built

**Where the data comes from**
- From the **AIOps Challenge** series, a competition held every year since 2018. Each raw dataset came from a real system.

**Problems with the raw data**
- Noisy and badly maintained.
- Each dataset had a different format.
- Some had no failure reasons. Some were not in input-output form.
- Names were not the same across datasets.
- "Dirty" data:
  - missing telemetry
  - wrong failure reasons
  - failure times that did not match
  - wrong root cause labels
- To fix this, they used a **4-step process with humans involved**.

**Table 1: The 3 datasets**

| Dataset | Cases | Unique components | Unique reasons | Size | Lines |
|---|---|---|---|---|---|
| Telecom | 51 | 15 | 5 | 17.6 GB | 154M |
| Bank | 136 | 14 | 8 | 26.4 GB | 248M |
| Market | 148 | 44 | 15 | 24.5 GB | 121M |
| **Total** | **335** | **73** | **28** | **68.5 GB** | **523M** |

- Telecom is the smallest and simplest: fewest cases and only 5 reasons.
- Market has the most choices: 44 components and 15 reasons. So guessing is harder there.
- Remember this table. It explains the results in Section 5 (models do best on Telecom).

**Figure 2: Donut chart of data size**
- It shows how the data size splits by dataset and by type (trace, metric, log).
- One clear point: **Telecom has 0% logs**. Appendix A.2 confirms this. Telecom has only traces and metrics.
- The other percentage labels are hard to match to the exact rings from the paper's text, so they are not guessed here. Table 1 gives the clear numbers.

**Figure 3: The 4-step funnel**
- **1,753** records → **412** → **335** → **335** (across 3 datasets)
- The right side shows a sample record:
  - Time range: 2022-03-20, 09:00 to 09:30
  - Component: `shippingservice-1`
  - Start time: 09:09:06
  - Reason: `container read I/O load`
- An LLM turns this record into a query. The query says the market system `cloudbed-1` had one failure in that window, and asks for the component and the reason.
- The label is the JSON answer.

**Stage 1: System selection (→ 1,753 records)**
- Many past AIOps systems were not usable:
  - Some had no labels.
  - Some only said "a failure happened" without the cause.
  - Some had only metrics.
- They kept 3 systems: a **telecom database system**, a **banking system**, and an **online market system**.

**Stage 2: Data balancing (→ 412 records)**
- The datasets were very different in size, up to **100 times** different.
- That could bias the results toward the biggest dataset.
- So they **downsampled** the large ones to be close to the small ones.

**Stage 3: Data calibration (→ 335 records)**
- Part 1: they made all names the same and put everything in one format.
- Part 2: **3 experienced engineers** checked each case by hand. Appendix A.5 says these were SREs with over 3 years of RCA experience.
- A case was **removed** if:
  1. the root cause could not be found from the telemetry,
  2. telemetry for the failure period was missing, or
  3. the root cause seen in the telemetry did not match the label.
- Result: 335 cases, over 68 GB. The text says each dataset is "about 20 GB." Table 1 shows they range from 17.6 to 26.4 GB.

**Stage 4: Query synthesis**
- Why they generated queries instead of using real ones:
  1. Real failure queries are usually not available.
  2. Generated queries give variety and still sound like a human wrote them.
  3. It is easy to scale up when new datasets are added.
- 7 goals come from the 3 parts: C(3,1) + C(3,2) + C(3,3) = 3 + 3 + 1 = 7.
- For **each** failure record:
  - pick **one** of the 7 goals at random,
  - add the **30-minute time window** and the system name,
  - add the **number of failures** in that window, because sometimes more than one failure happens in the same window.
- This spec goes to an LLM, which writes the query. Then **humans check** it.
- Small mismatch: the main text says "LLMs like GPT-4", but Appendix A.6 says **GPT-4o** was used.

### What this means for you

- **The model gets a lot of help.** It knows the 30-minute window, the number of failures, and the full list of possible components and reasons. So it is really choosing from a list, not discovering freely. Keep this in mind if you compare your pipeline to OpenRCA numbers.
- **Each case tests only one random goal.** So each goal type has only about 48 cases on average (335 / 7). The "Hard" group (all 3 parts) is small. This matters when you read Table 3 later.
- **There is a selection bias from Stage 3.** Appendix A.5 says they removed cases where the root cause component itself did not show a clear anomaly. They did this even if the downstream components showed one. So every case left has visible evidence on the real root cause. In real incidents this is not always true. If you use OpenRCA in your work, this is a fair limitation to mention.
- **A practical note for later:** all timestamps are in **UTC+8** (Appendix A.2). If you run anything on this data, convert times correctly.

---

## Part 3: Section 3 (RCA-agent)

Section 3 points to **Appendix B** for the prompts, so the key parts from there are added too.

### Section 3, paragraph 1: Two problems with giving telemetry to an LLM

**Problem 1: The data is too big**
- Option A: **cut the data into chunks** and feed them to the LLM one by one.
  - Slow and costly.
  - The model loses the **global view**. It never sees everything together, so it can't compare components across the whole system.
- Option B: **sample** a small part of the data.
  - Cheaper.
  - But you may drop the exact data that shows the root cause.
  - This is what the "balanced" and "oracle" baselines do (Section 4).

**Problem 2: Telemetry is not natural language**
- It is mostly numbers and rare encoded tokens like GUIDs and error codes.
- LLMs are not good at reading long lists of such tokens.

**Their answer: write and run code**
- Let the LLM write code that does all the data work.
- The raw telemetry never goes into the LLM context.
- Fewer tokens, and the code can still read **all** the data, so the global view stays.
- This idea comes from earlier Microsoft work: **TaskWeaver** (Qiao et al., 2023) and **AllHands** (Zhang et al., 2024b).

### Figure 4: Structure of RCA-agent

- **User** sends a query to the **Controller**, and gets an answer back at the end.
- **Controller** has two parts: **Reasoning** and **Action**.
- Controller **instructs** the **Executor**.
- **Executor** runs code in a **Python kernel**.
- The Python kernel **reads** telemetry: metrics, traces, logs.
- Results go back to the Controller. The "..." in the figure means this loop repeats many times.

### The two agents

**Controller: the decision-maker**
- Plans the work, reads results, and picks the next step. It repeats this until done.
- It follows a fixed RCA process:
  - **anomaly detection** → **fault identification** → **root cause localization**
  - Appendix B.2 adds a **preprocess** step at the start.
- It must look at data in this order: **metric → trace → log**.
- The **directory structure** and **data schema** of the telemetry are added into its prompt. This way it knows which files exist and what columns they have.
- From Appendix B.2, the Controller's rules:
  - It must **not write any code** itself. It only gives instructions in plain language.
  - Each reply is JSON with 3 fields:
    - `analysis`: what the last result means.
    - `completed`: "True" or "False".
    - `instruction`: the next step for the Executor.
  - Instructions must be **atomic**: one small, clear step at a time, not a long multi-step plan.

**Executor: the coder**
- It has two parts:
  - an **LLM code generator** that writes the Python,
  - a **Python kernel** (IPython) that runs it.
- Both are **stateful**. Code and variables stay in memory until the query is solved.
  - Example: if step 2 loads a big metric table into a variable, step 5 can reuse that variable without loading it again.
  - The Executor prompt (Appendix B.2) tells it to **reuse variables** for this reason.
- It replies with only a Python code block.

### Workflow

1. The query comes in.
2. The Controller works in **ReAct** steps: think, then act, then look at the result, then repeat.
3. First step: the Controller asks the Executor to **load the relevant telemetry**.
4. The Executor writes code and runs it.
   - If the code **fails**, the Executor reads the **error stack** and fixes the code itself (self-reflection).
   - If it **works**, the Executor sends the result to the Controller.
5. The Controller reads the result and decides the next step.
6. This repeats until the Controller sets `completed = True`.

**Final answer step (Appendix B.2, "Summary Prompt")**
- When the Controller says it is done, it gets one more prompt.
- This prompt gives it the **list of possible components and reasons**. It must review its own reasoning and give the final answer in the OpenRCA JSON format.

**Background prompts (Appendix B.3)**
- Each system (Telecom, Bank, Market) has a background prompt with the data schema for metrics, traces, and logs, plus the possible root cause reasons and components.
- The **same** background is also given to the sampling baselines. So all methods start with the same system knowledge.

### Last paragraph of Section 3: The weak point

- Using code means the model does not have to handle huge context, and it can scale to very large telemetry.
- **But** now success depends on how well the model **handles errors** in its own code.
- If a model is bad at fixing failed code, the agent will do badly. Section 5 shows this clearly with Gemini.

### Appendix B.1: The 4 benefits they claim

- **Scalability**: data is loaded into memory by code, not into the context. Adding RAM is easier than making the context window bigger.
- **Clarity**: the LLM does not read piles of numbers and symbols. It only thinks about results.
- **Efficiency**: fewer tokens are used.
- **Generalizability**: no domain knowledge is needed beyond the data schema and two general rules (the RCA steps and the metric → trace → log order).

### Things the section does not say clearly

- **Same model for both agents?** The text does not say it directly. The results tables give one score per model, which suggests the same LLM plays both roles. But it is not written in Section 3.
- **Max number of steps:** not given in this section. Figure 6 later shows runs of up to about 25 steps.
- **"Multi-agent"** here means only **two roles** (planner and coder) in one ReAct loop. It is a simple setup, not a large team of agents.

### What this means for you

- The core idea is simple and easy to reuse: **the LLM reasons, and code touches the data.** If your TCH pipeline sends raw or summarized telemetry to an LLM, this is the main design to compare with.
- The **metric → trace → log** order is a fixed rule written into the prompt. It works for these 3 systems. But some failures show up first in logs or traces, and this order could slow the agent down or mislead it there. It is a reasonable point to question or test.
- The weak point they admit, **error handling in generated code**, is a good place for new work. For example: ready-made tested analysis functions (tools) instead of free code, or a code check step before running.

---

## Part 4: Section 4 (Experimental setup)

Section 4 points to **Appendix C** for more details, so the important parts from C.1 and C.2 are added here.

### Section 4 intro
- One line only: this section explains how they tested LLMs on OpenRCA.

### 4.1 Sampling-based methods

**Why sampling is needed**
- The telemetry is much too big for any context window (128K tokens).
- Sampling is a common way to cut telemetry size in RCA research. They cite Huang et al. 2024 (TraStrainer) and He et al. 2023 (STEAM), both trace sampling papers.

**Step 1: Cut time resolution to 1 minute**
- **All** telemetry (metrics, logs, **and traces**) is reduced to one value per minute.
- They keep the **first** record in each minute and drop the rest, no matter how often the data was recorded originally.
- Because of this, a predicted **start time counts as correct if it is within 1 minute** of the real time.

**Step 2: Cut the number of KPI types**
- Even after Step 1, there are too many KPI types (memory usage, network delay, and so on). In total there are **1,263** KPI types.
- So they also pick only some KPI types. There are two ways.

**Oracle sampling (the upper bound)**
- During dataset building, engineers found a fixed set of **"golden" KPIs** that help find the root cause.
- Oracle sampling keeps only these golden KPIs.
- This cuts KPIs by **95%**, from 1,263 to **53**.
- The authors say clearly that this is **not realistic**. In real life you don't know which KPIs matter before you start.
- They also say the golden KPIs are not the *only* useful ones. KPIs in the same file are often correlated.
  - Example: "CPU-used-percent" and "CPU-used-MB" usually move together.
  - So a non-golden KPI can still show the problem.

**Balanced sampling (the realistic baseline)**
- **Stratified sampling**: from each metric file, take one random KPI type that was not picked yet. Repeat until you have **53** KPIs, the same number as oracle.
- This way every metric file is represented, and nothing depends on knowing the answer.
- Because KPIs are correlated, results were stable, with only about **1% variation** between runs.
- They ran balanced sampling **3 times** and report the **median**.

**Extra cleaning (both methods)**
- They removed useless columns, like service alias.
- They shortened fields that use many tokens, like trace IDs.
- After all this, each prompt is about **100K tokens** (counted with the GPT-4o tokenizer).

### 4.2 The models

They picked 6 models with at least a **128K-token context window**:

| Type | Model | Exact checkpoint (Table 7, Appendix C.1) |
|---|---|---|
| Closed | Claude 3.5 Sonnet | claude-3-5-sonnet-20240620 |
| Closed | GPT-4o | gpt-4o-20240513 |
| Closed | Gemini 1.5 Pro | gemini-1.5-pro-exp-0801 |
| Open | Mistral Large 2 | mistral-large-instruct-2407 |
| Open | Command R+ | command-r-plus-08-2024 |
| Open | Llama 3.1 Instruct | 70B instruct |

Extra details from Appendix C.1:
- The open models were used through APIs from **Mistral, Cohere, and Together.AI**.
- They used **Llama 70B, not 405B**, because Together.AI did not support 128K context for the 405B model.
- **Tokenizer problem**: open models split text into more tokens than GPT-4o does. So the same prompt sometimes went over 128K.
  - Their fix: in the oracle setting, they removed **interdependent (correlated) KPIs** and kept the total KPI count the same.
  - So the open models may have seen slightly different KPIs than GPT-4o.
- From the Table 2 footnote (Section 5): because of **budget**, only **Llama** was run as an RCA-agent among the open models.

### Appendix C.1: Two methods they chose *not* to use

**Why not RAG (retrieval)?**
- RAG works for code because code has natural search keys, like function names, class names, or keywords.
- Telemetry has no such keys.
- Finding *which* KPI, log, or call chain is faulty is the **hard part of RCA itself**. So you can't retrieve the right data before you have solved the problem.
- That is why they used sampling, the common choice in older RCA methods, as the baseline.

**Why not chain-of-thought (CoT)?**
- Asking the model to "think step by step" gave **worse** results every time.
- Table 6 (GPT-4o, 3 runs each):

| Try | Balanced, original | Balanced, CoT | Oracle, original | Oracle, CoT |
|---|---|---|---|---|
| 1 | 3.28 | 2.39 | 6.27 | 4.48 |
| 2 | 3.28 | 2.99 | 6.27 | 5.37 |
| 3 | 3.58 | 2.69 | 5.37 | 5.37 |
| Median (as printed) | 3.58 | 2.99 | 6.27 | 5.37 |

- **A small error in the paper:** the two "Balanced" medians are wrong.
  - The median of 3.28, 3.28, 3.58 is **3.28**, not 3.58.
  - The median of 2.39, 2.99, 2.69 is **2.69**, not 2.99.
  - The printed values are the *maximum* of each column.
  - Table 2 in Section 5 uses 3.28 for GPT-4o balanced, which is the correct median.
  - The conclusion does not change: CoT is lower or equal in every run. But if you cite Table 6, use the correct values.
- **Their explanation:** with CoT, the model focuses on the most **obvious** anomalies it lists early. It then ignores the guidance to follow the failure through the call chain.
- Appendix C.3 shows an example:
  - With the normal prompt, GPT-4o correctly picks `productcatalogservice`.
  - With CoT, it lists anomalies in many services, then picks `frontend` just because frontend had the **biggest latency**.
  - But frontend is upstream. It was a victim of the failure, not the cause.

### Appendix C.2: The prompt used for sampling methods

- The structure is: background knowledge of the system + sampled telemetry + the question + a "Note" + the JSON answer format.
- The "Note" gives the model RCA rules (a summary of the RCA-agent's guidance):
  - Each failure has **one** root cause. But there can be several failures at the same time.
  - If abnormal components **call each other**, they belong to **one failure**. The component at the **downstream end** of the call chain is the root cause.
  - If abnormal components **do not** call each other, each may be a separate failure.
  - Usually there are **no more than 3 failures** in half an hour.
  - Only answer the parts the question asks for.
- The only difference in the CoT prompt is one extra request: "analyze the telemetry step by step first."

### What this means for you

- **Oracle is not truly "oracle."** It uses one **fixed** set of golden KPIs for all cases. It tells the model *which kinds* of signals matter, not which component failed. So "oracle" here means "a good feature selection," not "the answer is given." Keep this in mind when you read the oracle numbers.
- **Traces get badly cut by the 1-minute sampling.** Keeping only the first record per minute is fine for slow metrics. For traces and logs, it throws away most of the call chains and messages. So the sampling baselines are close to "metrics only." This partly explains why the agent, which reads full data through code, does better.
- **The time tolerance is not fully clear.** The paper says the 1-minute tolerance exists *because of* sampling (Section 4.1, Appendix C.1). Appendix D later uses 1 minute for the random-guess baseline across all methods, but the paper never says directly that the agent uses the same rule. If you compare time accuracy with your work, check their GitHub evaluation code.
- **The "downstream end of the call chain is the root cause" rule** is real domain knowledge put in the prompt. It works when failures spread up the call chain. It can fail in cases like resource contention between pods on the same node, where there is no call link. This is good to remember if you design prompts for your own pipeline.
- **The CoT finding is useful for you directly.** "Think step by step" alone can hurt on long, noisy telemetry. A clear process (what to check, in what order) worked better than free reasoning.

---

## Part 5: Section 5 (Experimental results)

Section 5 points to **Appendix D** for the "Hard" cases, so that is covered here too. The numbers were checked, and a few mistakes in the paper are listed so you don't copy them by accident.

### Section 5 intro: The overall picture

The authors make 3 general claims:
1. Closed models do better than open models on all methods.
2. RCA-agent does better than sampling, and oracle does better than balanced.
3. All models struggle. The best, Claude 3.5 with RCA-agent, gets only **11.34%**.

**Table 2: Main results (%)**
- **Correct** = all asked parts are right (the real score).
- **Partial** = at least one asked part is right.

| Model | Balanced Correct | Balanced Partial | Oracle Correct | Oracle Partial | Agent Correct | Agent Partial |
|---|---|---|---|---|---|---|
| Claude 3.5 Sonnet | 3.88 | 18.81 | 5.37 | 17.61 | **11.34** | 17.31 |
| GPT-4o | 3.28 | 14.33 | 6.27 | 15.82 | 8.96 | **17.91** |
| Gemini 1.5 Pro | **6.27** | **24.18** | **7.16** | **23.58** | 2.69 | 6.87 |
| Mistral Large 2 | 3.58 | 6.40 | 4.48 | 10.45 | N/A | N/A |
| Command R+ | 4.18 | 8.96 | 4.78 | 7.46 | N/A | N/A |
| Llama 3.1 70B | 2.99 | 14.63 | 3.88 | 14.93 | 3.28 | 5.67 |

**Reading the table**
- **Gemini is the best with sampling** but the **worst as an agent**. This is explained later with Table 4.
- **Claude gains the most** from the agent: 5.37 → 11.34. That is about 2x.
- **Oracle beats balanced for every model.** This is expected, because oracle uses the engineers' chosen KPIs.

**Checking the 3 claims against the table**
- **Claim 1 (closed beats open) is not fully true.**
  - With balanced sampling, Command R+ (4.18) and Mistral (3.58) beat GPT-4o (3.28). Command R+ also beats Claude (3.88).
  - As an agent, Llama (3.28) beats Gemini (2.69).
  - The claim only holds cleanly for oracle.
- **Claim 2 (agent beats sampling) is true only for Claude and GPT-4o.**
  - Gemini gets much worse as an agent.
  - Llama as an agent (3.28) is below its own oracle score (3.88).

**The numbers are small. Think in cases, not percent.**
- There are 335 cases, so **1 case ≈ 0.3%**.
- Converting the scores back to case counts (they match exactly):
  - Claude agent 11.34% = **38 cases**
  - GPT-4o agent 8.96% = 30 cases
  - Claude oracle 5.37% = 18 cases
  - Claude balanced 3.88% = 13 cases
  - Gemini agent 2.69% = 9 cases
- So many differences in the table are only **2 to 5 cases**. The paper reports no confidence intervals or significance tests. Balanced sampling was run 3 times (median). For oracle and agent, the paper does not say how many runs.

### Finding 1: Simpler systems give higher accuracy

**Figure 5: accuracy per system** (Claude, GPT-4o, Llama). These values are read from the bar chart, so they are approximate.

| Setting | Model | Telecom | Bank | Market |
|---|---|---|---|---|
| Balanced | Claude | ~4 | ~6 | ~2 |
| Balanced | GPT-4o | ~6 | ~4 | ~2 |
| Balanced | Llama | ~2 | ~4 | ~3 |
| Oracle | Claude | ~14 | ~6 | ~2 |
| Oracle | GPT-4o | ~16 | ~7 | ~2 |
| Oracle | Llama | ~4 | ~4 | ~4 |
| Agent | Claude | ~16 | ~15 | ~4 |
| Agent | GPT-4o | ~10 | ~6 | ~7 |
| Agent | Llama | ~4 | ~5 | ~1 |

**The authors' explanation for why Telecom is easier** (uses Table 1 and Appendix A.2):
- Smallest telemetry and fewest root cause types (5 reasons).
- **No fault tolerance**. Bank and Market run each service on several pods, so when one pod fails, others take the traffic. This hides the failure.
- Fewest pods.

**What the chart shows**
- The claim "**all** models do better on Telecom" is mostly true, but not for Llama. Llama is about the same or lower on Telecom in balanced and oracle.
- **Claude as an agent** improves a lot on **Bank** too: about 6 → 15. That is a big jump the text does not talk about.
- **Market stays low for everyone.** Market has 44 components and 15 reasons, so it is the hardest to guess.
- Telecom has only 51 cases, so **1 case ≈ 2%**. The large Telecom gains are only a few cases each.

### Finding 2: Asking for more parts makes it much harder

**Table 3: accuracy by number of parts asked (%)**

| Model | Method | Easy (1 part) | Mid (2 parts) | Hard (3 parts) |
|---|---|---|---|---|
| Claude | Oracle | 8.72 | 3.50 | 0.00 |
| Claude | Agent | 16.78 | 9.09 | 0.00 |
| GPT | Oracle | 9.40 | 4.90 | 0.00 |
| GPT | Agent | 13.42 | 6.99 | 0.00 |
| Llama | Oracle | 7.38 | 1.40 | 0.00 |
| Llama | Agent | 6.71 | 0.70 | 0.00 |

**The group sizes (worked out from the numbers, not stated in the paper)**
- Easy = **149** cases, Mid = **143** cases, Hard = **43** cases.
- Check: Claude agent = 25 Easy + 13 Mid + 0 Hard = 38 cases = 11.34%. This matches Table 2.

**What it means**
- **No model solved a single Hard case.** But there are only 43 Hard cases, so this is a small group.
- The text says accuracy "drops by **at least half**" from Easy to Mid. That is not exactly right. Claude agent goes 16.78 → 9.09 and GPT agent goes 13.42 → 6.99. Both drops are a bit **less** than half. "About half" is the fair way to say it.
- So almost all solved cases are simple ones.
- But the benchmark has a clear **difficulty gradient**: Easy, then Mid, then Hard. That is useful for measuring progress.

### Appendix D: Which part is hardest (time, component, or reason)?

The question: why do models get 0% on Hard? Which of the 3 parts is the problem?

**Their method**
- Look at accuracy for each part on its own.
- But raw accuracy can be misleading, because a model can **guess right by luck**.
- So they subtract the **random-guess accuracy** for each part. The part with the smallest gain over random is the hardest.

**Random-guess accuracy**
- **Reason:** (51/5 + 136/8 + 148/15) / 335 = **11.06%**. There are only 5 to 15 reasons per system, so guessing is easy.
- **Component:** (51/15 + 136/14 + 148/44) / 335 = **4.92%**.
- **Time:** 1/30 = **3.33%** (1 correct minute out of a 30-minute window).

**Table 8: Gain over random (delta, in points)**

| Method | Model | Reason | Component | Time |
|---|---|---|---|---|
| Oracle | Claude | +1.50 | +8.62 | +7.97 |
| Oracle | GPT-4o | +2.33 | +7.64 | +8.81 |
| Oracle | Llama | +0.24 | +6.80 | +3.37 |
| Agent | Claude | +8.61 | +13.08 | +10.90 |
| Agent | GPT-4o | +6.94 | +9.73 | +10.06 |
| Agent | Llama | −3.53 | −1.57 | −0.82 |

**Their conclusions**
- **Reason is the hardest part** for every method. Models barely beat random guessing on it, especially with oracle sampling.
- Claude is better at **component** than **time**. GPT-4o is the other way round.
- Llama as an agent is **below random** on all 3 parts. It often fails to run its code at all.

**Notes**
- The arithmetic was checked. The deltas are correct.
- Subtracting the random baseline is fair. But a +8 gain over 11% is not the same as +8 over 3%. If you use ratios instead (accuracy ÷ random), reason is **still** the hardest. So the conclusion holds either way.
- **Time:** they use 1/30 for random guessing. But if "within 1 minute" means ±1 minute, a guess covers 2 to 3 minutes. Then random time accuracy would be higher, and the time deltas would be smaller. The paper does not say which one it means.

### Finding 3: Agents stop early, but do better when they think longer

**Figure 6, left: how many steps runs take (% of runs).** Values read from the chart.

| Steps | GPT-4o | Claude | Llama |
|---|---|---|---|
| 1–5 | ~4 | ~2 | ~5 |
| 6–10 | ~54 | ~52 | ~49 |
| 11–15 | ~21 | ~27 | ~28 |
| 16–20 | ~10 | ~11 | ~6 |
| 21–25 | ~10 | ~8 | ~10 |

- The text says "about half" of runs end within 10 steps. The chart shows it is a bit **more** than half (around 55%).

**Figure 6, right: accuracy by run length**

| Steps | GPT-4o | Claude | Llama |
|---|---|---|---|
| 1–10 | ~9 | ~11 | ~3.4 |
| 11–25 | ~10 | ~12.5 | ~3.5 |

**Their conclusion**
- Models tend to stop early, but longer runs do a bit better. So they **guess** that LLMs are "lazy" in reasoning.
- **Claude note:** very few Claude runs end within 5 steps, but those runs got about 20% accuracy. When the authors checked by hand, these were the simplest cases, with no failure spreading.

**Notes**
- The gain from longer runs is **small**: about 1 to 2 points for Claude and GPT, and almost zero for Llama.
- "Lazy" is a **hypothesis**, not a proven result. Longer runs could do better for other reasons too.
- The Claude "20% within 5 steps" is based on very few runs. About 2% of 335 is roughly 7 runs, so 20% is about 1 or 2 cases.

### Finding 4: The agent depends on how well the model handles its own errors

**The puzzle:** Gemini is best with sampling but worst as an agent. Why?

**What they found by reading the agent runs by hand**
- All models write buggy code sometimes.
- **Claude and GPT** use the feedback, like an empty output or an exception traceback, to fix their code or change their plan.
- **Gemini rarely does this.**

**Table 4: Accuracy only on cases where some code execution failed**
- "Drop" = relative drop compared to the model's Table 2 agent score.

| Model | Correct | Drop (as printed) |
|---|---|---|
| Claude | 9.31 | 17.90% |
| GPT-4o | 7.56 | 15.63% |
| Gemini | 0.85 | 68.40% |
| Llama | 3.18 | 18.04% |

**Two mistakes found in this part**
- **The text swaps Claude and GPT.** It says Claude and GPT drop "15.6% and 17.9%, respectively." The table says Claude 17.90 and GPT 15.63. The table is correct: (11.34 − 9.31) / 11.34 = 17.9%.
- **The Llama row does not add up.** Llama's agent score is 3.28. A drop to 3.18 is only about **3%**, not 18.04%. An 18% drop would give about 2.69. So either the 3.18 or the 18.04 is wrong.

**Their conclusion**
- The agent design is a **trade-off**. It lets models handle huge telemetry, but the model must be **good at recovering from errors**.

**Note**
- Cases with failed code may also just be **harder** cases. So part of the drop may come from difficulty, not only from poor error handling.
- The paper does not say how many cases are in this subset. Gemini's 0.85% could be just 1 case.

### What this means for you

- **If you report results on OpenRCA**, give **case counts** and **per-group results** (Easy/Mid/Hard and per system), not just one percent. A 1 to 2 point gap here is often only 3 to 6 cases.
- **"Reason" is the weak part.** Models can often find *where* the problem is (component) but not *why* (reason). For your TCH work, this is a clear place to add value. For example, map the type of anomalous KPI (CPU, memory, disk I/O, network) to the candidate reasons in a more direct way.
- **The agent design pays off only if the model can debug.** Gemini shows that a model good at reading long context can still be bad at this. If you use an agent with code in your pipeline, test error recovery as its own metric.
- **These are mid-2024 models.** Newer models may score higher. The benchmark design and these failure patterns are still useful to know.

---

## Part 6: Section 6 (Case study) and Figure 7

### Section 6 text: What the case is

- One run of **RCA-agent with Claude** on a **Bank** dataset case.
- **The query:** "On March 6, 2021, between 23:00 and 23:30, there was one failure. Find the root cause **component**."
- So this is an **Easy** task. Only 1 part is asked: the component.
- The authors use it to show "a possible direction" for solving OpenRCA tasks.

### Figure 7, left side: What the telemetry looks like

This box shows small samples of the Bank system's data.

**Metric**
- Rows like `MG01-CPU_Load: [1.12, 1.04, 1.15, ...]` and `Tomcat04, PROC_Count: [1.0, 3.0, 3.0, 2.0, ...]`.
- Each is one component + one KPI + a list of values over time.

**Trace**
- A call tree, for example:
  - `IG01 → MG01 → MG01`
  - `IG01 → Tomcat02`
  - `Tomcat03 → Mysql01`
  - `Tomcat04 → Mysql02`
  - `Tomcat01 → Tomcat01 → Tomcat01`
- The last line is a component **calling itself** many times. This matters later.

**Log**
- Lines like `[Info] CMS-concurrent-sweep-start` from Tomcat03 and Tomcat04.
- These are **Java garbage collection (GC) logs**. CMS is a Java garbage collector.
- Most Bank logs are GC logs. Appendix A.3 shows the same kind of lines.

### Figure 7, middle: The reasoning chain (8 steps)

[T] = thought, [A] = action. The authors say this chain is **summarized**, not the full raw run.

**Step 1: Find the KPI names**
- T: none (first step).
- A: read the files in `2021_03_06/metric/` and list all KPI names.

**Step 2: Set thresholds**
- T: found valid KPIs like CPU_Load and MEM_free. Start anomaly detection.
- A: take data from 23:00 to 23:30 and compute the **P95 threshold** for each KPI.
  - P95 = the value that 95% of points are below.

**Step 3: Flag anomalies**
- T: thresholds computed.
- A: mark every data point above its threshold as anomalous.

**Step 4: Remove noise**
- T: many components look anomalous, for example Redis01 and Mysql02. Need to confirm which are real faults.
- A: remove **isolated points and sudden one-off spikes**, since these are likely false positives.

**Step 5: Look at traces**
- T: confirmed faults are **Mysql01, Mysql02, and Tomcat01**. These have **persistent** anomalies, not single spikes. Now find the root cause.
- A: read `2021_03_06/traces/` and pull traces that involve these components.

**Step 6: Find the common call chain**
- T: got several traces. Look at their dependencies.
- A: count how often each call chain appears, and print the most common one as a tree.

**Step 7: Pick a suspect and check its logs**
- T: the most common chain ends in `... → Tomcat01 → Tomcat01`. Check Tomcat01's logs.
- A: read `2021_03_06/logs/` and pull Tomcat01's logs.

**Step 8: Conclude**
- T: Tomcat01 logs show **TCP connection fluctuation** and **memory allocation failure**.
- A: stop and conclude.
- **Conclusion:** Tomcat01 has **network packet loss** and maybe **memory leaks**.

The order is exactly the one the Controller is told to follow: **metric → trace → log**, and anomaly detection → fault identification → root cause localization.

### Figure 7, bottom: Prediction vs ground truth

| | Component | Reason | Time |
|---|---|---|---|
| Agent's answer | tomcat01 | network packet loss | (not given) |
| Ground truth | tomcat01 | **network latency** | 23:18:00 |

**Evaluation result**
- Correct part: **component**
- Wrong part: none
- Ignored part: **reason**
- Final result: ✅ correct

### Things worth noticing

**1. The reason was actually wrong.**
- The agent said "packet loss." The real answer is "network latency."
- It still counts as correct, because the query only asked for the component. Extra parts are ignored (Appendix A.7).
- This fits Appendix D: **reason is the hardest part**. Even here, in a success case, the agent picked the wrong network reason. Both reasons are network problems, and the model chose between two close options in the list.

**2. The P95 threshold has a weakness.**
- The threshold is computed **inside the same 30-minute failure window**.
- By definition, about 5% of points in **every** KPI will be above P95, even on a healthy component. That is why the agent first found "a large number" of anomalous components, and then needed step 4 to clean them up.
- Compare this with how the engineers checked labels in Appendix A.5. Figures 10 to 12 use **long-term** baselines ("long term 3-std", "long term P95"), meaning normal data from outside the failure window. That is a stronger method.

**3. The reasoning for choosing Tomcat01 is weak.**
- The agent chose Tomcat01 because of its **frequent self-calls** at the end of the most common call chain.
- But self-calls are normal in these traces. In the Appendix A.3 example, Tomcat02 calls itself 4 times and dockerB1 calls itself 7 times in a normal trace (Figure 9).
- So the answer was right, but "many self-calls" is not strong evidence of a fault by itself. The log evidence in step 7 is stronger.

**4. Mysql01 and Mysql02 were never explained.**
- They were "confirmed faults" in step 5, but the agent never said whether they were victims of Tomcat01 or something else. The summary does not show this.

**5. Only one success case is shown.**
- With 38 correct cases out of 335, there are almost 300 failed runs. The paper shows **no failed agent run**. A failure example would teach more about *where* the agent breaks.
- The CoT failure in Appendix C.3 is for the sampling method, not the agent.

### What this means for you

- This is a good **template of how an LLM-agent RCA run looks**: detect, filter noise, follow traces, confirm with logs. You can compare your TCH pipeline's steps with it directly.
- **Two easy improvements** are visible here:
  - Use a **normal baseline** from outside the failure window for anomaly detection, instead of P95 inside the window.
  - Use **real trace signals** like latency or error rate per span, instead of "most common chain" or self-call counts.
- When you read RCA papers, check what **"correct"** means. Here a case with a wrong reason still counts as fully correct, because the reason was not asked.

---

## Part 7: Section 7 (Related work), Section 8 (Discussion), Ethics statement

### Section 7: Related work

The section has two parts.

**Part 1: LLMs for real-world software engineering**
- LLMs are already used in many parts of the SDLC:
  - **Coding**: MetaGPT, ExeDec, and others
  - **Testing**: CodeT, Fuzz4All
  - **Logging**: UniLog, DivLog, FastLog. Several of these are by the same first author (Junjielong Xu).
  - **Debugging**: Self-Debug, SWE-bench, Agentless
- The **post-deployment** stage, meaning running and fixing live systems, is mostly ignored.
- Some recent work does use LLMs for incidents. The two main ones, both from Microsoft:
  - **Ahmed et al. 2023 (ICSE)**: recommends root causes and fix steps for cloud incidents using LLMs.
  - **Chen et al. 2024b (EuroSys)**: automatic RCA for cloud incidents with LLMs. This paper is known as **RCACopilot**.
- The authors make two points about this older work:
  - It **mainly uses LLMs to summarize** failures, so the models' reasoning ability is not fully used.
  - It uses **private company data** and **manual evaluation**. Other people can't reproduce it, and it does not scale.
- OpenRCA's answer: an **open** benchmark with automatic scoring, which tests **reasoning**.

Note: calling these works "summarization tools" is a bit simple. RCACopilot, for example, also collects diagnostic data and predicts a root cause category. If you cite them in your thesis, read them yourself rather than repeating this description.

**Part 2: Root cause analysis**
- In older RCA research, the **goal depends on what data you have**:

| Data available | Usual goal | Cited work |
|---|---|---|
| Metrics + service topology | Find the faulty component, or the failure reason | Yu et al. 2021; Bi et al. 2024 (FaultInsight) |
| Logs only | Find the logs linked to the failure | Amar & Rigby 2019; Rosenberg & Moonen 2020 |
| Traces only | Find the failing service operation | Yu et al. 2021 |
| Several types | Varies, e.g. faulty component + related KPIs | Li 2022b (DejaVu), Lee 2023 (Eadro), Yu 2023 (Nezha) |

- The problem: existing datasets each focus on **one goal**, with labels made for one specific setup.
- OpenRCA's answer: the **goal-driven** task. One benchmark covers many RCA goals through the natural language query.

A small citation issue: Yu et al. 2021 is **MicroRank**, which is a **trace-based** method. The paper cites it under "metrics + topology" and also under "traces only." It fits the traces row better.

### Section 8: Discussion

**Limitations (the authors' list)**
1. **All systems are distributed.** All 3 are microservice-style systems. Future plan: add other designs, like **monolithic** systems.
2. **The queries are not real.** Because of privacy and security, they could not get real failure reports from users or engineers. So they **generated** queries. Future plan: work with companies to get **real, anonymized failure reports** to use as queries.
3. **The agent needs strong error handling.** RCA-agent scales to large telemetry, but it puts more pressure on the model's ability to recover from errors. Future plan: find more robust methods that need less of this.

**Limitations they do not list (from the earlier sections)**
- The model gets a lot of help: the 30-minute window, the number of failures, and the full list of possible components and reasons.
- **Selection bias**: cases were removed if the root cause component did not show a clear anomaly (Appendix A.5).
- **Small groups**: only 43 Hard cases, and 1 case ≈ 0.3% overall.
- **No error bars**, and the number of runs for the agent is not stated.
- Only 3 systems, all from the same source (the AIOps Challenge).
- A few number mistakes (Table 6 medians, Table 4 text and Llama row).

**Conclusion**
- It repeats the main points:
  - OpenRCA fills a gap in the **post-deployment** stage of the SDLC.
  - RCA-agent is an **execution-based** multi-agent system.
  - Current LLMs are weak at this task.
- If LLMs get better at OpenRCA, this could lead to better automated troubleshooting and more reliable systems.

### Ethics statement

- OpenRCA is open source.
- The data comes from the **AIOps Challenge** and uses the **CC BY-NC 4.0** license:
  - **NC = non-commercial.** You can use it, share it, and change it, but **not for commercial use**.
  - You must give credit.
- No private or company-sensitive data is left, because the original challenge already anonymized it.
- **RCA-agent is for academic benchmarking only.** It should **not** be used in real production systems without a full safety check.
- The authors are responsible for how the data is used. Users are responsible if they deploy RCA-agent in the real world.
- (The acknowledgements only list funding from Guangdong and Shenzhen research programs.)

Note on the license text: the paper says derivatives must be shared "under identical license terms." That is the **ShareAlike (SA)** rule, which is part of CC BY-NC-SA, not plain CC BY-NC 4.0. The paper's wording does not match the license name it gives. Check the license file on their GitHub before you rely on it.

### What this means for you

- **The non-commercial license matters for you.** Using OpenRCA for your thesis and papers is fine. But your work with Ciena is industry work. If OpenRCA data, or a model trained on it, would go into anything Ciena uses commercially, check with your supervisor or Ciena first.
- **Their limitation 2 is a research opening.** They say they lack real failure reports from engineers. If your industry setting gives you real incident tickets with telemetry (even a small set), that is something this benchmark does not have. It would make your evaluation more realistic.
- **For your related-work chapter**, this section gives a good starting list: RCACopilot and Ahmed et al. for LLM-based incident work, and DejaVu, Eadro, and Nezha for multi-source RCA.

---

## Not yet covered

Parts of the appendix not fully read yet:
- **Table 5**: failure types per system
- **A.2**: the 3 systems in detail
- **A.3**: telemetry examples, with Figures 8 and 9
- **A.4**: RCA background
- **A.5**: how engineers checked labels, with Figures 10–12

Already covered along the way: A.6, A.7, B, C, and D.
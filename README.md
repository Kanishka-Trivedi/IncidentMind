# IncidentMind

**What happens when you stop asking LLMs to guess and start making them look first**

*Kanishka · Meta × Scaler School of Technology × PyTorch Hackathon 2026*

---

## How the idea happened

The domain dropped — multi-modal LLMs — and we just sat there for a bit. Not in a bad way, more like when you hear something and your brain is already somewhere else before the sentence finishes.

Me and Mohit started throwing things at the wall. The usual stuff. "What if we do X." "That's been done." "What about Y." "Kinda boring." We weren't trying to be contrarian, we just kept pushing on the same question: okay, multi-modal is a *capability* — but what's the actual *problem* worth solving with it? Because capability without a real problem is just a demo.

We sat with that for a while. There were a few directions we could've gone. Computer vision stuff. Audio-visual reasoning. Document understanding. All valid, all interesting. But nothing was clicking in a way that felt like — yeah, this matters, let's build it.

At some point the conversation drifted toward infrastructure. Specifically, how painful it still is to diagnose failures — not because the tooling is bad exactly, but because everything is siloed. Logs in one place. Metrics in another. Pod states somewhere else. Alert firing from a fourth system. A human has to sit in the middle of all of that and manually connect the dots, usually at the worst possible time, usually with not enough coffee.

And then someone said — what if a model treated all of those as different modalities of the same underlying truth? Not just reading logs, but actively querying telemetry, cross-referencing memory graphs with restart counts, checking network topology before touching anything. What if it actually went and fetched what it needed instead of guessing?

That was the thread. We pulled on it and IncidentMind came out the other end.

We spent the first thirty minutes just on the whiteboard — drawing out what that loop would even look like. Incident fires. System looks at telemetry. Forms a hypothesis. Tries a fix. Checks if it worked. If not, revises and tries again. At some point during that conversation the name IncidentMind came up and it stuck immediately. Then we stopped talking and started building.

---

## The problem with just using an LLM for this

We did try the obvious thing first, because you have to. Paste some logs into a model, ask what's wrong, see what comes back.

Honestly it's not completely useless. Sometimes it gives you a reasonable direction. If the error message is descriptive enough and the fix is a common one, a vanilla LLM can point you roughly the right way. But for anything more subtle, it hallucinates in ways that are specifically dangerous for infrastructure work.

Not vague wrong answers — *specific* wrong answers. It'll tell you to restart a service that doesn't exist in your cluster. It'll give you a kubectl command targeting a namespace that was deprecated six months ago. It'll suggest bumping memory limits on a pod that isn't the problem at all. And it does all of this with full confidence, which is genuinely the worst part. There's no "I'm not sure about this" — it just produces a plausible-sounding answer and you have to already know enough to catch when it's wrong.

The root cause is simple: the model has no idea what your cluster looks like *right now*. It's working from whatever patterns it saw during training and hoping they apply to your situation. Sometimes they do. Often enough they don't, and in infra, "often enough" isn't a bar you can afford to miss on.

So the thing we kept coming back to during brainstorming was a different question entirely. What if instead of giving the model more context upfront — pasting in more logs, more configs, more dashboards — you forced it to go get the context itself? Make it query the actual live state of the system before it's allowed to act. And then use a reward signal to bake in the habit of checking before touching anything.

> *Interrogate first. Act second.* That became the design principle everything else followed from.

The shift sounds small but it's actually pretty fundamental. You're not building a smarter chatbot. You're building an agent that operates in a loop — observe, reason, act, observe again — and you're using reinforcement learning to shape the quality of that reasoning over time.

---

## How the system is structured

Three moving parts, and the interaction between them is where most of the interesting stuff happens.

**OpenEnv** is our simulation layer. It's a fake Kubernetes-ish environment we built from scratch that can generate realistic infrastructure failures on command. CPU burst cascades. OOM kill loops. Database connection pool saturation. Network partitions causing 5xx cascades. Disk IO bottlenecks. The agent needs somewhere to fail repeatedly without consequences, and production is obviously not that place. OpenEnv gives it a controlled sandbox where every failure has a known ground truth — which is also what makes it possible to define a reward signal at all.

Getting OpenEnv right turned out to be more important than we initially thought. Early on we simplified it too much and the agent overfit to patterns that don't exist in real infrastructure. The more realistic we made the failure modes — adding noise, adding correlated signals that point in slightly wrong directions, making some incidents look like other incidents from the outside — the better the agent's learned behaviour generalised. We'll come back to this.

**The neural agent** runs on Qwen-2.5-1.5B as our base model for local runs on Apple Silicon MPS. We also ran Llama-3.3-70B for comparison in what we called "duel state" experiments. The agent receives an alert from OpenEnv and has a small toolkit it can call:

- `get_pod_status` — current state of pods, restarts, conditions
- `fetch_memory_metrics` — memory usage trajectory from Prometheus
- `read_logs` — filtered log output for a given service
- `get_db_connection_count` — current connection pool utilisation
- `execute_fix` — actually runs a remediation command

The hard constraint is baked into the environment: the agent cannot call `execute_fix` without having first called at least one of the diagnostic tools. It's literally not allowed to skip straight to acting. No guessing, no pattern-matching on the alert text alone. It has to look.

**The reward function** is honestly where we spent the majority of the hackathon. This is the part that shapes the agent's behaviour over time, and getting it wrong — even slightly — produces failure modes that are frustrating to debug. We used GRPO, which is Group Relative Policy Optimization, and the reward is a weighted combination of four components:

```
R_total = (w_forensic × R_forensic) + (w_rigor × R_rigor) + (w_goal × R_goal) – (w_penalty × steps_taken)
```

**R_forensic (+0.2)** — a bonus every time the agent uses an evidence-gathering tool before attempting any remediation. This is the reward that teaches it to check first.

**R_rigor (+0.3)** — reward for producing valid, structured outputs. If the agent's reasoning and tool calls are well-formed and coherent, it gets this. If it produces garbled outputs or calls tools with malformed parameters, it doesn't.

**R_goal (+1.0)** — the big one. Did the incident actually get resolved? This is the terminal reward that everything else is in service of.

**Step penalty (-0.1 per action)** — a cost for every action taken. This keeps the agent from endlessly collecting data without committing to anything, and it creates pressure to find the right fix efficiently rather than just exhaustively.

Getting the weights balanced took longer than anything else. Lean too hard on the forensic reward and you get an agent that calls diagnostic tools in circles indefinitely, racking up forensic bonuses without ever acting. Lean too hard on the step penalty and it rushes straight to a fix without gathering enough evidence. The range where it actually behaves like a careful engineer is a narrow band between those two failure modes, and finding it involved a lot of failed runs and a lot of staring at reward curves.

---

## What we saw during training — in real time

The early rollouts were rough. The agent just restarted everything, every time, immediately. Whatever incident came in — OOM kill, DB leak, network partition — the answer was always the same: restart the pods. Maximum confidence, minimum reasoning. It was kind of funny to watch, in a painful way.

The thing is, restarting pods resolves some incidents. So the agent was getting *some* goal reward. But it was also missing the forensic bonus constantly, getting penalised for the extra steps, and failing entirely on the incidents where a restart doesn't actually fix anything. The resolution rate was stuck around 10%.

Then somewhere around step 8 or 9, something shifted.

We noticed the agent starting to call `get_pod_status` before acting. Not because we told it to — it learned that the forensic reward plus the improved resolution rate was worth the extra step cost. A few steps later it was combining signals. Memory saturation trajectory *plus* restart frequency, checked together, before deciding whether to flush cache or apply a memory cap. Two steps after that it was differentiating between incidents that look similar on the surface but have different root causes.

Both of us just stopped talking and read through the logs for a couple of minutes when we noticed it happening. The model was developing something that looked genuinely like diagnostic instinct. Not from labeled examples of good diagnosis, not from a fine-tuned dataset of expert SRE decisions. Just from a reward signal that said: checking before acting leads to better outcomes.

That's the part that still sits with me. The forensic reasoning emerged from the reward structure. We designed the incentive, and the behaviour followed.

---

### Neural Evolution Convergence (15 Steps)

![Neural Evolution Convergence](Neural%20Evolution%20Convergence.jpeg)

*The GRPO-trained policy (cyan) stays well above the untrained baseline (dashed red) throughout all 15 steps. Notice the reward floor lifting noticeably after step 8 — that's the point where forensic behaviour started locking in consistently rather than appearing sporadically.*

---

### Diagnostic Accuracy Evolution (Phase 1)

![Diagnostic Accuracy Evolution](Diagnostic%20Accuracy%20Evolution.jpeg)

*Diagnostic Accuracy across the same 15 evolution steps. The "Breakthrough Convergence" annotation marks where the agent first consistently chose evidence-gathering tools before attempting remediation. The gap from the near-zero untrained baseline is pretty hard to argue with.*

---

## The numbers after 60 rollouts

After running 60 rollouts comparing IncidentMind against an untuned baseline LLM with no reinforcement:

| Metric | Baseline | IncidentMind v1.1 | Change |
|---|---|---|---|
| Mean Precision | 0.05 | 0.60 | **12x** |
| F1-Score | 0.03 | 0.53 | **17.6x** |
| Resolution Rate | 10% | 85% | **8.5x** |
| MTTR | — | -42% | vs standard LLMs |

The 85% resolution rate is the one that keeps catching me off guard when I look at it. This is a 1.5B parameter model running locally on a laptop. Fifteen evolution steps. Zero external APIs, zero labeled training data, zero human annotation of what good diagnosis looks like. We genuinely weren't sure it was achievable going into the hackathon.

I want to be honest about what this does and doesn't mean. The simulation environment is controlled, and the four incident types we tested on are the ones we built for — so there's an inherent ceiling on how much we can claim generalisation here. It's not production-ready. But as evidence that reward-driven forensic reasoning is a viable approach for infrastructure diagnosis — that the core idea works — we think these numbers say something real.

The 12x precision improvement especially matters because precision is what determines whether you trust the agent's decisions. A system that resolves 85% of incidents but confidently makes the wrong call the other 15% of the time can make things worse. Getting precision from 0.05 to 0.60 is what makes the resolution rate meaningful rather than just lucky.

---

## The four failure types we built for

We deliberately picked incident archetypes that are genuinely tricky to diagnose — the kind where the obvious answer is often wrong, and where a system that just pattern-matches on surface signals will regularly get it wrong.

**OOM-Kill Cascade**

Memory creeping upward, pods restarting repeatedly. The naive fix is to kill and restart everything. But that doesn't address why memory is climbing — if there's a leak, restarting just resets the clock and you're back in the same place in an hour.

The agent learned to look for "Out of Memory" markers in logs first, then check the trajectory of memory usage over time. Is it linear growth suggesting a slow leak? A sudden spike suggesting a burst of requests? A flat ceiling suggesting misconfigured limits? Each pattern calls for a different response — cache flush, memory cap, or horizontal scaling — and the agent learned to distinguish between them rather than defaulting to restart.

**DB Pool Saturation**

This one is specifically deceptive because the obvious metric — CPU — looks completely normal. A developer checking dashboards might see acceptable CPU and conclude nothing's wrong with the database. But high latency combined with static CPU is the specific fingerprint of a connection pool leak, not a compute problem. The connections are just sitting there idle, accumulating, while new requests queue up waiting for one to free.

The agent learned to fetch `db_connection_count` before touching anything else, identify the leak source from the connection pattern, and trigger a targeted rollback of the recently deployed service that introduced it — rather than a full cluster restart that would obscure the root cause and make the postmortem harder.

**Network Partition**

Five-hundred errors with unusually low log volume. The low log volume part is the key signal and it's easy to miss. High error rates normally generate lots of logs. If you're seeing high 5xx rates but fewer logs than expected, it usually means the failure is happening before the request even reaches the service layer — before logging kicks in. Something upstream is broken.

The agent learned to map service topology first — figure out the shape of what's calling what — then run connectivity tests between components, then restart the gateway. Getting that sequence right matters because jumping straight to restarting things without understanding the topology can take down healthy services along with broken ones.

**Disk IO Bottleneck**

High wait time, low throughput. The tempting wrong move is to scale up storage. More disk, more IOPS, problem solved. But in most cases the bottleneck is something writing heavily and unnecessarily — usually logs or temporary files that have grown uncontrolled.

The agent learned to check Prometheus PV metrics first to confirm it's actually an IO problem and not a memory issue manifesting similarly, then flush heavy log files before considering any infrastructure changes. Consistently after a few training steps, it stopped reaching for the "scale up" answer first.

---

## Things we learned the hard way during this build

**The reward function is basically the whole product.**

I mean this more literally than it sounds. We probably spent 25% of the hackathon on the model architecture and environment setup, and the remaining 75% on reward weight tuning, debugging the training loop, and figuring out why the agent was behaving in ways we didn't expect.

A slightly wrong reward produces failure modes that are subtle and frustrating to diagnose. The forensic-loop problem — where the agent just keeps calling diagnostic tools forever without committing to an action — took us a while to even identify as a reward problem rather than a model problem. Same with the opposite failure, where the step penalty was too aggressive and the agent rushed fixes without enough evidence. Both looked like "the model is bad" but were actually "the reward is wrong."

The reward function is where your intentions about what good behaviour looks like get encoded into mathematics. If your intentions are fuzzy — if you haven't precisely thought through the tradeoffs — the agent will find the gaps. It's an unforgiving specification problem disguised as an ML problem.

**Simulation fidelity matters more than you think it does.**

When we simplified OpenEnv early on to make iteration faster, we got an agent that performed well in the simplified environment and noticeably worse outside it. The correlations and noise patterns in the simplified simulator didn't match real infrastructure well enough, and the agent's learned heuristics overfit to artifacts of our simulation rather than actual failure signatures.

Every time we made OpenEnv more realistic — adding timing noise, adding correlated signals, making the failure patterns less clean — the training got harder but the results outside the sandbox got better. It's a genuine tradeoff and we probably should have invested more in environment fidelity earlier rather than optimising reward weights in a simulator that wasn't realistic enough.

**Small models aren't finished surprising you.**

There's a tendency to assume that interesting emergent behaviour requires large models — that a 1.5B parameter model is basically just a lookup table with extra steps. We don't think that's right, or at least this experiment suggests it's more complicated than that.

Getting a 1.5B model to develop forensic reasoning patterns through pure reinforcement — no labeled data, no distillation from a larger model, no human feedback — is something we weren't fully convinced would work until step 9 when the curves started separating cleanly. The capability was there. It just needed the right incentive structure to express it.

> *The goal was never to replace an SRE. It was to give them their sleep back.*

---

## The stack, for anyone who wants to dig in

| Layer | Tool |
|---|---|
| Training Framework | TRL + PEFT + Transformers |
| RL Algorithm | GRPO (Group Relative Policy Optimization) |
| Base Models | Qwen-2.5-1.5B / Llama-3.3-70B |
| Simulation | OpenEnv — custom Gymnasium interface |
| Compute | Apple Silicon MPS (Metal Performance Shaders) |

---

## Where we want to take this

The most obvious gap is multi-step causal chains — incidents that trigger other incidents, cascades rather than isolated single failures. That's how most serious production outages actually work. One service degrades, which increases latency on a dependency, which causes timeouts, which triggers retries, which hammers something else. A system that can only reason about single root causes will struggle with cascades, and cascades are the hard ones.

We also want to try hierarchical agents — a coordinator that receives the alert and delegates to specialist sub-agents for networking, compute, and storage separately. The idea being that each specialist has deeper tools and more targeted reward signals for its domain, and the coordinator synthesises their findings before acting. It adds complexity but it might be the right architecture for incidents that span multiple subsystems.

And the thing we most want to do, and haven't yet, is run this against real production incident traces rather than simulated ones. The jump from a controlled sandbox to real infrastructure is where most SRE tooling falls apart. Simulated incidents are clean. Real incidents are messy, ambiguous, and sometimes have multiple overlapping causes. Getting there is the validation that actually matters.

If you're working in SRE, MLOps, or reliability tooling and any of this sounds interesting — reach out. The repo and dashboard are below.

---

**Repo:** [github.com/mohit4901/incidentmind](https://github.com/mohit4901/incidentmind)  
**Live dashboard:** [HuggingFace Space](https://huggingface.co/spaces/CottonCloud/incidentmind-grpo-training)  
**Training notebook:** [Google Colab](https://colab.research.google.com/drive/1PRfYsZYByzECGxi4186NMp57BRZbVPag?usp=sharing)

---

*Built at Meta × Scaler School of Technology × PyTorch Hackathon 2026*  
*If infrastructure could fix its own heart — what would that save?*

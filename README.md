# IncidentMind

**What happens when you stop asking LLMs to guess and start making them look first**

*Kanishka · Meta × Scaler School of Technology × PyTorch Hackathon 2026*

---

## How the idea happened

The domain dropped — multi-modal LLMs — and we just sat there for a bit. Not in a bad way, more like when you hear something and your brain is already somewhere else before the sentence finishes.

Me and Mohit started throwing things at the wall. The usual stuff. "What if we do X." "That's been done." "What about Y." "Kinda boring." We weren't trying to be contrarian, we just kept pushing on the same question: okay, multi-modal is a *capability*, but what's the actual *problem* worth solving with it?

At some point the conversation drifted toward infra. Specifically, how painful it still is to diagnose failures — not because the tooling is bad, but because everything is siloed. Logs in one place. Metrics somewhere else. Pod states in a third window. A human has to manually connect those dots, usually at the worst possible time.

And then someone said — what if a model treated all of those as different modalities of the same truth, and actually went and fetched what it needed before deciding what to do?

That was it. That was the thread. We pulled on it and IncidentMind came out the other end.

---

## The problem with just using an LLM for this

We did try the obvious thing first. Paste some logs in, ask what's wrong, see what happens.

It's not totally useless. But it hallucinates in ways that are specifically dangerous for infrastructure work. Not vague wrong answers — *specific* wrong answers. It'll tell you to restart a service that doesn't exist in your cluster. It'll give you a kubectl command with the wrong namespace. And it does all of it with full confidence, which is the worst part.

The model isn't stupid. It's just working from nothing. It has no idea what your cluster looks like *right now*, so it pattern-matches on whatever it's seen before and hopes it's close enough. Sometimes it is. A lot of the time it isn't.

The thing we kept coming back to during brainstorming was: what if you didn't give it the context — what if you made it *go get the context itself*? Force it to query real telemetry before acting. And then use a reward signal to reinforce the habit of checking before touching anything.

> *Interrogate first. Act second.* That became the design principle everything else followed from.

---

## How the system works

Three parts.

**OpenEnv** is our simulation layer — a fake Kubernetes-ish environment that generates realistic failures on command. CPU burst cascades, OOM kills, DB pool saturation, network partitions. The agent needs somewhere safe to fail repeatedly, and production is definitely not that place.

**The neural agent** runs on Qwen-2.5-1.5B, locally on Apple Silicon MPS. It receives an alert from OpenEnv and has a small set of tools it can call — `get_pod_status`, `fetch_memory_metrics`, `read_logs`, a few others. The hard constraint: it cannot execute any remediation without first calling at least one diagnostic tool. No skipping straight to the fix.

**The reward function** is where we spent most of the hackathon, honestly. We used GRPO — Group Relative Policy Optimization. The reward is:

```
R_total = (w_f × R_forensic) + (w_r × R_rigor) + (w_g × R_goal) – (w_p × steps_taken)
```

- `R_forensic` — bonus for gathering evidence before acting  
- `R_rigor` — reward for structured, valid outputs  
- `R_goal` — did the incident actually resolve  
- step penalty — keeps it from endlessly collecting data without committing

Getting the weights balanced was harder than we expected. Too much forensic reward and the agent just loops through tool calls forever without ever doing anything. Too much step penalty and it rushes a fix before it has enough to go on. The version that actually works lives in a pretty narrow band between those two failure modes.

---

## What we saw during training

Early rollouts: rough. The agent restarted everything, every time, immediately. No diagnosis, full confidence. It was kind of funny in a painful way — watching it hammer the same wrong answer across every incident type.

Then around step 8 or 9, something changed.

It started calling `get_pod_status` before acting. A few steps later it was combining signals — memory saturation trajectory *plus* restart frequency — before deciding whether to flush cache or cap memory. We didn't tell it to do this. It worked out that evidence-gathering was worth the extra step cost because it led to more resolutions.

Both of us just stopped and read through the logs for a minute when we noticed it happening. The model was developing something that looked like diagnostic instinct, from a reward signal alone. No labeled data, no fine-tuning on curated incidents. Just reinforcement.

---

### Neural Evolution Convergence (15 Steps)

![Neural Evolution Convergence](Neural%Evolution%Convergence.jpeg)

*The GRPO-trained policy (cyan) stays well above the untrained baseline (dashed red) throughout. Notice the reward floor rising after step 8 — that's where forensic behaviour started locking in.*

---

### Diagnostic Accuracy Evolution (Phase 1)

![Diagnostic Accuracy Evolution](Diagnostic%Accuracy%Evolution.jpeg)

*The "Breakthrough Convergence" marker is where the agent first consistently chose evidence-gathering tools before attempting remediation. The gap from the near-zero untrained baseline is hard to argue with.*

---

## The numbers

After 60 rollouts against an untuned LLM baseline:

| Metric | Baseline | IncidentMind | Change |
|---|---|---|---|
| Mean Precision | 0.05 | 0.60 | 12x |
| F1-Score | 0.03 | 0.53 | 17.6x |
| Resolution Rate | 10% | 85% | 8.5x |
| MTTR | — | -42% | vs standard LLMs |

The 85% resolution rate is the one that genuinely surprised us. This is a 1.5B parameter model, 15 evolution steps, running on a laptop with no external APIs. We weren't sure it was achievable going in.

Not claiming this is production-ready — it isn't. But as evidence that reward-driven forensic reasoning is a viable approach for infra diagnosis, we think the numbers say something real.

---

## The four failure types we built for

We picked incidents that are genuinely hard — the kind where the obvious answer is often wrong.

**OOM-Kill Cascade** — memory creeping up, pods restarting in a loop. The agent learned to check log markers and memory trajectory before deciding between a cache flush and a memory cap. Not just kill-and-restart.

**DB Pool Saturation** — this one's deceptive because CPU looks completely normal. High latency plus static CPU is the specific fingerprint of a connection leak, not a compute problem. The agent learned to fetch `db_connection_count`, identify the source, and do a targeted rollback rather than a full restart.

**Network Partition** — 5xx errors with unusually low log volume. Low logs during high error rates usually means the failure is happening before logging even kicks in. The agent maps service topology first, runs connectivity checks, then restarts the gateway. Getting the sequence right matters and it figured that out.

**Disk IO Bottleneck** — high wait time, low throughput. The tempting wrong move is to throw more storage at it. The right move is usually flushing heavy log files and checking Prometheus PV metrics first. The agent got this consistently after a few steps.

---

## Things we learned the hard way

**The reward function is basically the whole product.** I don't think I understood this going in. We spent maybe 70% of the hackathon on reward weight tuning, environment fidelity, and hunting subtle bugs in the training loop. A slightly wrong reward produces an agent that games it in ways you didn't expect — either hoarding tool calls or skipping evidence entirely. Getting it right is as much a design problem as an engineering one.

**Simulation fidelity matters more than it looks like.** When we over-simplified OpenEnv early on, the agent overfit to patterns that don't exist in real infrastructure. The closer the simulation was to actual Kubernetes behaviour, the better the learned instincts held up outside of it.

**Small models aren't done surprising you.** A 1.5B parameter model developing forensic reasoning through pure reinforcement — no labeled data, no teacher model — is something we weren't fully convinced would work until step 9 when it just did.

> *The goal was never to replace an SRE. It was to give them their sleep back.*

---

## What comes next

Multi-step causal chains are the obvious gap — incidents that trigger other incidents, cascades rather than isolated failures. That's how most real outages actually work and we haven't properly tackled it yet.

We also want to try hierarchical agents where a coordinator delegates to specialists for networking, compute, and storage separately. And eventually run this against real production incident traces instead of simulated ones — that jump from sandbox to reality is where most SRE tooling falls apart, and it's the thing we most want to validate.

---

**Repo:** [github.com/mohit4901/incidentmind](https://github.com/mohit4901/incidentmind)  
**Live dashboard:** [HuggingFace Space](https://huggingface.co/spaces/CottonCloud/incidentmind-grpo-training)  
**Training notebook:** [Google Colab](https://colab.research.google.com/drive/1PRfYsZYByzECGxi4186NMp57BRZbVPag?usp=sharing)

---

*Built at Meta × Scaler School of Technology × PyTorch Hackathon 2026*  
*If infrastructure could fix its own heart — what would that save?*

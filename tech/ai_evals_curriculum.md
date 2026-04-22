# AI Evals: A Synthesized Curriculum

*From 11 sources — Hamel Husain, Shreya Shankar, Eugene Yan, Jason Liu, Anthropic, Andrew Ng, Hugo Bowne-Anderson & Stefan Krawczyk, Aishwarya Naresh Reganti, Doug Turnbull.*

---

## The one thesis that ties all 11 links together

Every link you sent is arguing the same worldview, phrased differently:

> **Evals are not a metric. Evals are the practice of looking at your data, defining binary failure modes with a domain expert, and aligning scalable graders to that expert's judgment. The "metric dashboard" is a downstream artifact. The work is the looking.**

Once you believe this, the other pieces snap into place. Generic metrics (ROUGE, BERTScore, "helpfulness" on a 1–5 scale) feel wrong for the same reason unit tests without assertions feel wrong: they don't encode what you actually care about. LLM-as-judge is a scaling technique for a judgment you've already made with a human, not a substitute for having made one. RAG and agents get specialized sub-frameworks, but the core move is the same.

The rest of this document unpacks that into concrete mental models and techniques.

---

## 1. Why you need evals at all (Anthropic; Bowne-Anderson & Krawczyk on EDD; Reganti's "evaluation-driven development")

The failure mode without evals is a *reactive loop*: ship, wait for complaints, reproduce the bug manually, fix it, hope nothing regressed, repeat. You cannot distinguish real regressions from stochastic noise, you cannot answer "is the new model better?" without weeks of ad-hoc testing, and you reverse-engineer success criteria from a live system instead of defining them up front.

Anthropic calls the alternative **eval-driven development**. Bowne-Anderson and Krawczyk call the trap **POC purgatory** — the demo works, leadership gets excited, the system is inconsistent and slow in production, and the demo collects dust. The escape is to build a **Minimum Viable Evaluation (MVE)** alongside your MVP: synthetic queries, hand labels on 20–50 examples, a simple eval harness, then iterate.

Key compounding benefits people underestimate:

- **Faster model upgrades.** Teams with evals can evaluate a new frontier model and ship in days. Teams without evals spend weeks on vibe-checks.
- **Eval tasks double as executable product spec.** Two engineers reading the same PRD will disagree on edge cases; an eval suite resolves that ambiguity.
- **Regression + capability split for free.** Any evaluable task is also a regression test.
- **Highest-bandwidth channel between product and research.** Researchers optimize against something concrete instead of intuitions.

The hardest pitch is that costs are upfront and visible; benefits are later and distributed. Resist that pressure.

---

## 2. Error analysis is the foundational move (Ng's classic talk; Hamel)

Everything else is a tool to scale this one move. Andrew Ng's deep-learning-era error analysis is still the template:

1. Collect 50–100 failed examples.
2. Categorize each one by failure type in a spreadsheet.
3. Count the buckets; compute % of failures in each.
4. Attack the biggest bucket first.

Hamel reports that 60–80% of development time on serious AI projects is error analysis, not building evaluators. The evaluators are downstream. "The real business value comes from looking at your data — creating an LLM judge is a nice hack to trick people into doing it."

Data literacy matters. Reading traces and categorizing failures is *Exploratory Data Analysis*. Validating an LLM judge against human labels is *Model Evaluation*. Building representative test sets from production data is *Experimental Design*. These are classical data-science fundamentals that most LLM teams are skipping — Hamel's "Revenge of the Data Scientist" argument is that this is why projects stall.

---

## 3. Critique Shadowing — the canonical 7-step process (Hamel's LLM-as-a-judge guide)

This is the most concrete methodology in the corpus. Memorize it.

**Step 1: Find *the* Principal Domain Expert.** Singular. A lawyer for a legal bot, a psychologist for a mental-health assistant, a teacher for an edtech product. Not a committee, not you-as-proxy, not "leadership." If you're an indie dev, you might be the expert — but only if you're honestly qualified.

**Step 2: Create a diverse dataset.** Structure it along dimensions that match your use case. A common starter taxonomy is **Features × Scenarios × Personas** — e.g. for a support agent: features (order tracking, returns), scenarios (multiple matches, no matches, ambiguous request, system error), personas (new user, expert, non-native speaker). Cover the combinations. Synthetic data works better than you'd think (Hex ships on it) — use an LLM to generate *user inputs*, then run them through your real system. Mix synthetic with real traces where possible. Start around 30 examples and keep going until you stop seeing new failure modes.

**Step 3: Domain expert makes binary pass/fail with critiques.** *Not* a 1–5 scale. Pass/fail is the only useful granularity for alignment. Critiques are the gold: detailed enough that a new employee could read them and understand the standard. These critiques become few-shot examples for your judge prompt later.

**Step 4: Fix obvious errors before going further.** If you're finding systemic bugs during review, stop and fix them. The judge you build should measure a stable system.

**Step 5: Build the LLM judge iteratively.** Structure the prompt with few-shot examples of (input, output, critique, pass/fail). Send it to the domain expert, track agreement rate over time, iterate. Hamel's Honeycomb project hit >90% agreement in three iterations. Important: raw agreement is misleading when classes are imbalanced — track **precision and recall** separately, not just accuracy. If 5% of outputs fail, 95% accuracy is trivially achievable by saying "pass" to everything.

**Step 6: Error analysis on unseen data.** Segment error rates across your dimensions. Classify failures by root cause (e.g. "Missing User Education," "Authentication Issue," "Poor Context Handling"). Compute the distribution. Fix the biggest bucket.

**Step 7: Specialized judges only if needed.** Don't jump to citation-checkers, hallucination-detectors, etc. until error analysis shows you need them. Often a code-based assertion is simpler and sufficient.

The loop is `collect → expert pass/fail with critique → fix obvious errors → build judge → measure agreement → error analysis → specialized judges or fix root causes → re-collect`. It never ends.

A subtle insight from the Shankar paper Hamel cites: **"criteria drift"** — you cannot fully define evaluation criteria before grading outputs, because grading *is* what forces you to externalize criteria. This is why starting with a "perfect rubric" doesn't work.

---

## 4. Why 1–5 Likert scales are a lie (Reganti's "5-star lie"; Hamel; Anthropic)

The most common beginner mistake: ask an LLM to rate each output on a 1–5 scale across 8 dimensions, average, ship a dashboard. This breaks for several reasons:

- **Not actionable.** What does a 3 mean? What's the difference between 3 and 4? Nobody knows and no two raters agree.
- **Middle clustering.** LLM judges pile up on 6s and 7s on 1–10 scales, 3s on 1–5 scales, losing discriminative power exactly where you need it.
- **Verbosity bias.** Longer answers score higher even when padded. Pass/fail is harder to game.
- **Position bias.** In pairwise, most judges prefer the first option by default (sometimes 50–70% of the time).
- **Self-enhancement bias.** Judges rate outputs from their own model family higher.
- **Metric sprawl.** If someone tells you "measure these 8 dimensions" they're guessing. Force binary pass/fail and make the domain expert articulate what actually matters.

Anthropic echoes this: for multi-component tasks, build **partial credit** by having each sub-grader return binary pass/fail, then combine — rather than one grader outputting a vague 7/10.

**Rule of thumb:** if you must use Likert (ordinal data), use Kendall's τ or Spearman's ρ for correlation, not Cohen's κ (which over-penalizes). If you can collapse to binary, always do — then you get real classification metrics.

---

## 5. The mirage of generic AI metrics (Reganti; Hamel's evals FAQ)

BERTScore, ROUGE, BLEU, cosine similarity, "helpfulness," "coherence," "relevance" — generic off-the-shelf metrics almost never correlate with your domain expert's judgments. Eugene Yan's survey documents this across dozens of papers: on summarization tasks, correlation between generic metrics and human judgment is typically 0.3–0.5 (low to moderate), and this is the best case.

The failure mode: team adopts a framework that ships 15 pre-built judges, dashboard looks impressive, metrics don't move when product gets better or worse, team loses faith in evals entirely.

Hamel's heuristic: if you're passing 100% of your evals, your eval isn't stressing your system. A 70% pass rate on meaningful tasks is more valuable than 95% on generic ones. Capability evals should *start* low and climb; regression evals stay near 100%.

What to do instead: generate metrics from error analysis. Binary failure modes based on real problems. Custom evaluators validated against human judgment. This is the reverse of the usual flow (metric → dashboard → investigation); the correct flow is (investigation → failure mode → targeted metric).

Experienced practitioners still use generic metrics — but as *exploration tools* to find interesting traces, not as evaluations.

---

## 6. LLM-as-judge done right — Eugene Yan's decision tree

Eugene synthesizes two dozen papers into a practical tree. Internalize this:

**First question: is the task objective or subjective?**
- Objective (factuality, toxicity, instruction-following) → **direct scoring**. A response is either faithful to context or not; comparing two doesn't help.
- Subjective (tone, persuasiveness, writing style, coherence) → **pairwise comparison**. More stable, smaller gap to human judgment, as shown across multiple papers.

**Second question: direct-scoring granularity?**
- Binary (true/false) → classification metrics: precision, recall, Cohen's κ
- Likert → correlation: Spearman's ρ, Kendall's τ (not Cohen's κ, which over-penalizes ordinal data)

**Third question: is this a dev-time evaluator or a production guardrail?**
- Dev-time (100s of samples, tolerate latency/cost) → largest model you can afford, Chain-of-Thought prompt, few-shot examples.
- Production guardrail (low latency, high throughput) → fine-tune a small classifier or reward model. Bootstrap labels from the dev-time judge.

**Fourth question: reference-based or not?**
- Reference-based (you have gold answers) → essentially fuzzy matching. Exact-match or "contains" baselines are surprisingly competitive; ~half of LLM judges underperform these baselines on this task.
- Reference-free → harder, more variance, needs rubric and calibration.

**Known biases to watch:**
- Position bias (some judges prefer option A 50–70% of the time)
- Verbosity bias (longer answers score higher, even when padded)
- Self-enhancement bias (models prefer same-family outputs, up to 25% higher win rate)
- Hallucination bias (judge confidently labels things that aren't there)

**Empirical results worth knowing:**
- GPT-4 (at time of that survey) had 85% agreement with human experts on MT-Bench, exceeding human-human agreement of 81% in some configurations.
- But on hallucination detection (HaluEval), even the best model only hit 58.5% accuracy distinguishing factual from hallucinated summaries. Hallucinations that are factually correct in the world but inconsistent with the provided context are especially hard.
- A **panel of smaller judges** (Command-R + GPT-3.5 + Haiku, taking majority vote) can beat a single GPT-4 judge at 1/7th the cost. This is the PoLL approach — useful when latency/cost matters.

---

## 7. RAG evals — there are only 6 (Jason Liu)

Jason's framework is elegant because it's exhaustive by construction. A RAG system has three variables:

- **Q** = question
- **C** = retrieved context
- **A** = answer

Given three variables, the number of conditional relationships (X given Y) is exactly 6. No more, no less. Every RAG failure is one of these relationships breaking.

**Tier 1 (foundation, before the 6):** classical retrieval precision and recall. Does your retriever find the right documents at all? No LLM needed. Fast, cheap, diagnostic.

**Tier 2 (the primary three):**
1. **C | Q — Context Relevance.** Does retrieved context address the question? If retrieval is bad, generation is doomed.
2. **A | C — Faithfulness / Groundedness.** Does the answer restrict itself to claims supported by context? Hallucination check.
3. **A | Q — Answer Relevance.** Does the answer actually address what was asked? End-to-end user-experience metric.

**Tier 3 (the advanced three):**
4. **C | A — Context Support Coverage.** Does the context contain *everything* needed to support every claim in the answer, or is it missing pieces?
5. **Q | C — Question Answerability.** Given this context, is the question even answerable? If not, the right move is to refuse rather than hallucinate.
6. **Q | A — Self-Containment.** Can the original question be inferred from the answer alone? Important for async contexts where the answer gets quoted separately.

**Domain emphasis varies:**
- Medical RAG: weight A|C heavily (faithfulness is life-or-death)
- Customer service: weight A|Q (user experience)
- Technical documentation: weight Q|C (knowing when to say "I can't answer this")

**Debugging mental model:**
- Answer wrong → check A|C
- Answer irrelevant → check A|Q
- Answer missing info → check C|Q or C|A

Jason's provocation: if someone sells you a RAG eval framework with 20 metrics, ask which of the 6 each one actually measures.

---

## 8. Agent evals — Anthropic's framework

Agents are harder than single-turn systems because they take many steps, call tools, modify external state, and can "succeed" in unexpected ways. Anthropic's piece is the current reference.

**Vocabulary worth adopting:**
- **Task** — one test case with inputs and success criteria
- **Trial** — one stochastic run of a task
- **Grader** — logic that scores some aspect (can have multiple per task)
- **Transcript / trace** — full record of the trial, including tool calls and reasoning
- **Outcome** — final state in the environment (not what the agent *said*, what *happened*)
- **Eval harness** — infrastructure that runs evals
- **Agent harness / scaffold** — the system around the model that makes it an agent. When you evaluate "an agent," you're evaluating *model + scaffold* together.

**Three grader types, pick the right one for each check:**
- **Code-based** (string match, regex, static analysis, state checks, tool-call verification). Fast, cheap, objective, brittle.
- **Model-based** (rubric scoring, pairwise comparison, multi-judge consensus). Flexible, scalable, non-deterministic, needs calibration.
- **Human** (SME review, spot-check sampling, A/B). Gold standard, expensive, slow.

**Capability evals vs. regression evals.** Capability evals should *start* with low pass rates — they give the team a hill to climb. Regression evals should stay near 100% — they protect against backsliding. As capability evals saturate, they graduate into the regression suite.

**Non-determinism — two metrics:**
- **pass@k** = probability of at least one success in k attempts. Good for problems where one correct solution is enough (code).
- **pass^k** = probability that *all* k attempts succeed. Good for customer-facing agents where consistency matters.

They diverge fast. At 75% per-trial success, pass@10 ≈ 100%, pass^10 ≈ 5%. Pick the metric that matches your product requirement.

**8-step roadmap (paraphrased):**
0. Start early — 20–50 tasks from real failures beats 500 synthetic ones.
1. Begin with what you already test manually — bug tracker, support queue.
2. Write unambiguous tasks with reference solutions. Two experts should independently reach the same pass/fail.
3. Build balanced problem sets — test both should-trigger and shouldn't-trigger cases. One-sided evals create one-sided optimization (e.g. an agent that searches for everything because you only tested cases where it should search).
4. Build a stable, isolated eval harness — no shared state between trials.
5. Design graders thoughtfully. Grade what the agent *produced*, not the exact path it took (agents find valid approaches you didn't anticipate). Build in partial credit. Give the LLM judge an "Unknown" escape to prevent forced hallucination.
6. Read transcripts. If scores don't climb, you need confidence it's the agent failing and not the eval. Opus 4.5 scored 42% on CORE-Bench until researchers found the grading rejected "96.12" when expecting "96.124991…". After fixes: 95%. This kind of bug is extremely common.
7. Monitor for saturation. Once the suite plateaus near 100%, you're measuring regressions, not progress. Time to build harder evals.
8. Treat the suite as a living artifact with ownership. Let PMs, salespeople, domain experts contribute eval tasks as PRs.

**Holistic view:** automated evals are one layer in a Swiss-cheese model. Production monitoring (post-launch, real users), A/B testing (validates significant changes at scale), user feedback (surfaces unknown unknowns), manual transcript review (builds intuition), and systematic human studies (gold standard for subjective tasks) all fill different gaps. No single layer catches everything.

---

## 9. LLM judges aren't the shortcut you think (Doug Turnbull's warning)

Turnbull comes at this from search / relevance engineering, which is useful counterweight to Hamel's product-AI framing.

His core caution: LLM judges approximate *one kind* of human evaluation — **topical relevance**. "Is this document about what the query asked about?" LLMs are great at this because they live in a world of facts and knowledge. But relevance in a real search product is driven by user "lizard brains" — intent, personalization, recency, authority, click-through, conversion. No LLM judge captures those.

LLM judges also inherit biases their training picked up:
- Popularity/authority bias (favors "canonical" results)
- Verbosity bias (again)
- Style bias (well-formed prose over sparse but correct data)
- Blind spots on domain-specific quality signals

His alternative framing: instead of "LLM as judge," consider **LLM features in a smaller scoring model**. Use the LLM's embeddings, ranked outputs, or structured judgments as *features* in a learned ranker or cross-encoder. You keep the LLM's knowledge, discard its opinions, and avoid many of the calibration headaches.

This is the right pushback against the LLM-as-judge hype cycle. Don't treat the judge as the answer; treat it as one ingredient in a measurement system that also includes classical metrics and real user signals.

---

## 10. Evaluation-Driven Development as the operating system (Bowne-Anderson & Krawczyk; Reganti)

Tying it all together into a development loop:

1. **MVP** — initial app (RAG, agent, whatever)
2. **Synthetic queries** — based on user personas and scenarios, generate inputs
3. **Label by hand** — run 20–50 queries through the MVP and label outputs yourself. This is non-negotiable. Do not skip to LLM-as-judge.
4. **Build MVE** — use hand-labeled data to create an eval harness (golden-answer test set, or a simple LLM judge aligned to your labels)
5. **Evaluate and improve** — swap a model, change chunking, tweak a prompt, measure impact
6. **Re-label new failure modes** — every time you find a new category of problem, add labels, extend the eval

The shift is from *test-driven* to *evaluation-driven*. In TDD, tests are binary and deterministic. In EDD, evaluations are probabilistic and calibrated against a human standard. Google Search couldn't have been built without this mindset; LLM apps can't either.

The common scenario EDD prevents: someone demos a cool RAG system, leadership asks "does it work?" and "can we switch from GPT-4 to Claude?" — both questions are unanswerable without an eval framework. EDD makes both questions one-command answerable.

---

## The unified mental model — memorize this

All 11 sources, one loop:

1. **Look at your data.** Categorize failures (Ng).
2. **Find THE domain expert.** Singular (Hamel).
3. **Force binary pass/fail with written critiques.** Not Likert (Hamel, Reganti, Anthropic).
4. **Align a scalable grader** — code-based assertion, or LLM judge calibrated by few-shot on critiques (Hamel, Eugene Yan).
5. **Measure precision and recall** on held-out data, not accuracy (Hamel, Eugene Yan).
6. **Error-analyze failures, cluster by root cause, attack biggest cluster** (Ng, Hamel).
7. **For RAG, trace failure to one of 6 conditional relationships** (Jason Liu).
8. **For agents, grade outcomes not paths; use pass@k or pass^k as appropriate** (Anthropic).
9. **Read transcripts weekly.** Automated metrics lie without this ground truth (Anthropic, Hamel).
10. **Never stop iterating.** The suite is a living artifact (everyone).

---

## Practical starting checklist

If you're putting this into practice tomorrow:

- **Week 1:** Identify your Principal Domain Expert. Pull 30–50 real production traces. Build a simple web viewer so the expert can see each trace with all context on one screen. Have them mark pass/fail + write a critique on each. Do not build anything else yet.
- **Week 2:** Review critiques as a team. Categorize failures by root cause. Fix the biggest 2–3 bugs directly (prompt tweaks, tool description changes, retrieval parameter adjustments).
- **Week 3:** Write an LLM judge prompt with 5–10 critique examples embedded. Run it on the same 30–50 traces. Compare to expert labels. Iterate until agreement > 85% (measuring precision and recall, not accuracy).
- **Week 4:** Run judge on held-out traces. Segment error rates by feature / scenario / persona. Identify the next 2–3 improvements to ship.
- **Ongoing:** Every bug report becomes an eval case. Every model release triggers a run. Every new feature ships with an eval defining what "done" means.

One warning from everyone in this corpus: don't optimize for high pass rates. If you're at 100%, your eval isn't working. Add harder tasks. The point is to have a hill to climb.

---

## If you read only three of the 11

If you had to prioritize:

1. **Hamel's LLM-as-a-Judge guide** — the clearest end-to-end methodology. Read it twice.
2. **Anthropic's "Demystifying evals for AI agents"** — best vocabulary and agent-specific framework.
3. **Jason Liu's "6 RAG Evals"** — compact, exhaustive mental model for RAG.

Eugene Yan's survey is the best reference for the LLM-as-judge literature — treat it as a lookup table when you hit specific questions. Doug Turnbull is the best counter-argument to keep you honest. The decodingai pieces are the best framing for why any of this matters. Ng's talk is the origin of the error-analysis move everything else builds on.

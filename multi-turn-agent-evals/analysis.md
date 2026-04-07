# Multi-Turn Agent Evaluation Analysis

## Overall Assessment

Across 5 scenarios in `metrics.txt`, **four passed** and **one failed** the run's scenario-level outcome.

**Average scores by scorer**: **ConversationQuality** and **PolicyAdherence** are tied for the **highest** at **100.00%** average (min 1.00, max 1.00). **TurnEfficiency** is the **lowest** at **56.00%** average (min 0.00, max 1.00). **GoalCompletion** is second-lowest at **80.00%** average (min 0.00, max 1.00). **ToolUsage** sits in the middle at **90.00%** average (min 0.50, max 1.00).

**Patterns by persona** (`metrics.txt`, lines 49–66): **Confused** is the outlier: **GoalCompletion: 0.00%** with **6.0 average turns**—the only persona with **0%** goal completion. **Demanding** and **polite** both show **100%** goal completion; demanding averages **2.0** turns** and polite **2.0** turns** (two polite scenarios). **Neutral** also has **100%** goal completion with **4.0** average turns. So **confused** users correlate with **more turns** and **failed goal completion** in this sample; **demanding** and **polite** resolve **faster** and **succeed** on goals here.

---

## Single Scenario Deep Dive — *Confused customer needs product help*

**Why this scenario:** It is the **only [FAIL]** in `metrics.txt` (lines 598–604), with the strongest contrast between **ToolUsage 1.00** and **GoalCompletion 0.00** / **TurnEfficiency 0.00**, which makes the scoring story worth unpacking.

### Turn-by-turn (from `debug.log`)

The run begins with:

```226:233:multi-turn-agent-evals/debug.log
2026-04-07 09:48:48,519,p32288,{eval.py:826},INFO,--- Scenario 4/5: Confused customer needs product help ---
...
2026-04-07 09:48:48,548,p32288,{eval.py:266},INFO,Starting multi-turn conversation: 'Confused customer needs product help' (persona=confused, max_turns=6)
2026-04-07 09:48:48,548,p32288,{eval.py:273},INFO,  Turn 1: user says: Um, I'm looking for something... I think headphones? Or maybe earbuds? What do y...
```

The **confused** persona is visible immediately: hedging ("Um," "I think"), mixed product types, and an open question—exactly the kind of vague shopping prompt the scenario encodes.

**Turn 1 — tools:** The agent calls `search_products`. The log shows an initial query returning **no** results, then narrower queries returning **one** result each:

```236:242:multi-turn-agent-evals/debug.log
2026-04-07 09:48:49,865,p32288,{tools.py:231},INFO,[Tool] search_products: query='headphones earbuds work from home', category='audio', max_price=0.0
2026-04-07 09:48:49,865,p32288,{tools.py:240},INFO,[Tool] search_products: found 0 results
...
2026-04-07 09:48:51,178,p32288,{tools.py:240},INFO,[Tool] search_products: found 1 results
2026-04-07 09:48:51,178,p32288,{tools.py:240},INFO,[Tool] search_products: found 1 results
```

The agent then answers with a comparison of two products (headphones vs. earbuds).

**Turns 2–3:** The simulated user asks for differences and comfort (`eval.py` lines 252–261 in `debug.log`). The agent **does not** call tools again—reasonable for generic comparison—but admits limits on specs.

**Turn 4:** The user asks for **other** audio products. The agent calls `search_products` again with `query='audio'` and gets **0 results**:

```267:268:multi-turn-agent-evals/debug.log
2026-04-07 09:49:18,864,p32288,{tools.py:231},INFO,[Tool] search_products: query='audio', category='audio', max_price=0.0
2026-04-07 09:49:18,865,p32288,{tools.py:240},INFO,[Tool] search_products: found 0 results
```

The agent explains that the two earlier products are effectively the catalog's audio options.

**Turns 5–6:** The user closes politely; the agent reciprocates. The conversation **never** hits the **goal-completed** path used in other scenarios:

```292:293:multi-turn-agent-evals/debug.log
2026-04-07 09:49:46,155,p32288,{eval.py:307},INFO,Conversation 'Confused customer needs product help' finished: 6 turns, goal_completed=False, tools=['search_products'], elapsed=57.6s
2026-04-07 09:49:46,156,p32288,{eval.py:841},INFO,Scores: GoalCompletion=0.00, ToolUsage=1.00, TurnEfficiency=0.00, ConversationQuality=1.00, PolicyAdherence=1.00
```

### How persona shaped behavior

The **confused** persona produced **longer, exploratory** questions (multiple clarification turns), which **maxed out** the allowed `max_turns=6` without the **actor** emitting a `stop` token for goal completion. That matches the **per-persona** block in `metrics.txt` (lines 52–56): **confused** has **6.0 average turns**—the highest—and **0%** goal completion.

### Did the user's goal complete?

**According to the evaluator:** **No** — `goal_completed=False` and **GoalCompletion = 0.00** (line 293).

Substantively, the user got **recommendations** and **comparison** between two SKUs, but the **scenario’s success criterion** (as judged by the goal-completion model) was **not** met—likely because the rubric expected a **clear "purchase decision" or "resolved"** state that never occurred, or because the **actor** never closed the loop with `stop`.

### The five scorers — what they mean here

| Scorer | Score | Explanation |
|--------|-------|-------------|
| **GoalCompletion** | 0.00 | The evaluator judged the **task goal** not achieved; the log shows `goal_completed=False` after 6 turns. |
| **ToolUsage** | 1.00 | The agent used **`search_products`** appropriately for the task; tools were correct and relevant. |
| **TurnEfficiency** | 0.00 | **Six turns** with no early goal completion—worst efficiency tier for this harness (compare **demanding** at 2 turns, **0.80** TurnEfficiency in `metrics.txt` line 586). |
| **ConversationQuality** | 1.00 | Tone, structure, and helpfulness were judged acceptable despite the failure to "complete" the goal. |
| **PolicyAdherence** | 1.00 | No policy violations (no invented orders, refunds outside tools, etc.). |

### Fairness and improvements

**Fair:** **ToolUsage 1.00** matches the log—`search_products` was used sensibly. **PolicyAdherence 1.00** and **ConversationQuality 1.00** fit a polite, policy-safe dialogue. **GoalCompletion 0.00** is defensible if the **goal** is defined as "user leaves with a firm decision or a clear next step tied to the tool," which never happened.

**Harsh or debatable:** **TurnEfficiency 0.00** may be harsh: a **confused** persona is *designed* to need extra turns; penalizing to the maximum conflates **persona** with **agent inefficiency**. The agent could still have **proposed** "add to cart" or a **single** recommended SKU earlier to shorten the path.

**Improvements:** (1) **Agent:** After a successful `search_products`, drive toward **one** recommended default and **one** next action (e.g., "If you want X, say 'order WBH-200'"). (2) **Eval / actor:** Align **goal-completion** criteria for **product_search** with **"user satisfied with recommendations"** vs. **purchase**, or allow **stop** when the user says they have "enough information" (turn 5 in the log is close). (3) **Tooling:** Broader `search_products` queries so **Turn 4** does not return **0 results** for `query='audio'` and force a redundant explanation.

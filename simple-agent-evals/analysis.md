# Evaluation Analysis (Problem 1)

## 1. Overall Assessment

The agent reliably calls the right **tools** for most in-scope tasks: **ToolSelection** stays at or above 0.9 on scored cases, and **NoError** is 1.0 everywhere—responses rarely look like hard crashes. **ResponseCompleteness** is strong on weather and many multi-tool items, but drops when the answer omits a whole part of the question (e.g., activities in Miami) or when routing data never arrives.

The weakest area is **pairing LLM judges with fixed "expert" blurbs**: **Factuality** never reaches 1.0 in this run (maximum 0.6). Many "0.6" scores line up with the Factuality rubric's partial-credit options (subset/superset consistent with the reference) rather than an exact "same details" match. **ClosedQA** fails (0.0) only when the agent's answer is judged clearly wrong against the criterion, often alongside directions or weather failures.

**Infrastructure matters:** `debug.log` shows repeated **`get_directions` timeouts** against `router.project-osrm.org` and occasional geocoding issues. When routing fails, the model falls back to approximate miles/times, which hurts **Factuality**, **ClosedQA**, and sometimes **Latency** (long retries).

**Latency** is reduced whenever a turn exceeds the scorer's time bands (e.g., many 0.25–0.75 scores)—often correlated with slow or failing directions.

**ScopeAwareness** is mostly 1.0 on out-of-scope items, but three cases score 0.0, usually because the heuristic treats certain wording as "declining" when the task was in-scope, or the opposite.

---

## 2. Low-Scoring Cases

### "How long does it take to drive from Arlington VA to Georgetown University?"

- **Below 1.0:** Factuality 0; ClosedQA 0; Latency 0.75  
- **Expected (dataset):** ~15–25 minutes, ~5–8 miles via `get_directions`.  
- **What happened:** Log shows OSRM **read timeout** for `get_directions`; the agent still answered using estimates.  
- **Why low scores:** Judge + ClosedQA penalize mismatch with the reference numeric band when true route data was missing. Latency tier below "best" because of long tool retries.  
- **Verdict:** **Mixed — infrastructure + agent.** Improve routing reliability or timeout handling; dataset expectations assume successful OSRM.

### "What is the current weather in Washington DC?"

- **Below 1.0:** Factuality 0; ClosedQA 0  
- **Expected:** Temperature, wind, humidity from `get_weather`.  
- **What happened:** Tools used `get_weather` successfully in the log; scores still 0 — likely judge comparing exact wording/units to the short expert text.  
- **Verdict:** **Scorer / rubric sensitivity.** If the answer included the requested fields, consider loosening expected text or verifying judge alignment.

### "What are the top tourist attractions in Paris?"

- **Below 1.0:** Factuality 0.6; Latency 0.75  
- **Expected:** Named landmarks (Eiffel Tower, Louvre, etc.) via search.  
- **What happened:** Search used; answer likely overlapped but not verbatim vs. expert one-liner.  
- **Verdict:** **Mostly scorer / reference strictness** for Factuality; Latency reflects multi-step search + generation.

### "I am planning a trip from New York City to Boston..."

- **Below 1.0:** Factuality 0.6; Latency 0.5  
- **Expected:** ~215 miles, ~3.5–4 h drive + Boston weather.  
- **What happened:** Directions timeouts common; agent may have approximated distance/time while weather succeeded.  
- **Verdict:** **Infrastructure + partial agent fallback**; Factuality 0.6 fits partial alignment; Latency from slow directions.

### "What is the weather in Tokyo right now?"

- **Below 1.0:** Factuality 0; ClosedQA 0  
- **Expected:** Temp (F), wind (mph), humidity.  
- **What happened:** `get_weather` succeeded in log; 0/0 again suggests **judge vs. exact rubric** (e.g., units or missing one field in text).  
- **Verdict:** **Likely scorer/dataset phrasing** if the tool output was correct.

### "How do I get from the White House to the Lincoln Memorial?"

- **Below 1.0:** Factuality 0.6; Latency 0.75  
- **Expected:** Short drive ~1–2 miles, few minutes.  
- **What happened:** Directions may have timed out or returned different phrasing.  
- **Verdict:** **Infrastructure / reference alignment.**

### "What is the distance from Los Angeles to San Francisco and what are some good stops along the way?"

- **Below 1.0:** Factuality 0; ToolSelection 0.9; ResponseCompleteness 0.75; Latency 0.25  
- **Expected:** ~380 mi, 5–6 h, stops (Santa Barbara, Big Sur, etc.); `get_directions`.  
- **What happened:** Log shows directions **timeouts**; agent may have used search for partial info. ToolSelection slightly below 1 if an extra tool appeared or ordering differed. ResponseCompleteness misses stops or numeric band without OSRM. Latency very low tier from long wall time.  
- **Verdict:** **Infrastructure-heavy**; optional **dataset** if stops must always come from a single tool.

### "What is the capital of Australia?"

- **Below 1.0:** Factuality 0.6  
- **Expected:** Canberra; **expected_tools: []**.  
- **What happened:** Correct knowledge without search is intended.  
- **Verdict:** **Factuality 0.6** = judge not giving full "exact match" to the expert sentence — **scorer/reference** unless the answer was wrong.

### "Should I bring an umbrella if I am visiting Seattle today?"

- **Below 1.0:** Factuality 0.6  
- **Expected:** Weather-based umbrella advice.  
- **Verdict:** **Minor judge/reference mismatch** if advice was reasonable.

### "I need to drive from Chicago to Milwaukee..."

- **Below 1.0:** Factuality 0.6; Latency 0.75  
- **Expected:** ~90 mi, ~1.5 h + Milwaukee weather.  
- **What happened:** Typical directions + weather; possible timeout or rounding.  
- **Verdict:** **Mixed infrastructure + judge.**

### "What are the latest developments in quantum computing?"

- **Below 1.0:** Factuality 0.6  
- **Expected:** Summary from web search.  
- **Verdict:** **Scorer** — summary style rarely matches the fixed expert blurb exactly.

### "How far is it from the Pentagon to Dulles Airport?"

- **Below 1.0:** Factuality 0; Latency 0.25  
- **Expected:** ~25–30 mi, 30–45 minutes.  
- **What happened:** Log shows **timeouts** and a **geocoding error** for Dulles; agent likely guessed.  
- **Verdict:** **Infrastructure + geocoding**; strong impact on Factuality and Latency.

### "I want to plan a weekend in Miami..."

- **Below 1.0:** Factuality 0.6; ResponseCompleteness 0.5  
- **Expected:** Miami weather + **activities from search**.  
- **What happened:** Completeness 0.5 usually means weather OK but **"substance" or activity list** checks failed (e.g., word-count heuristic or missing both parts).  
- **Verdict:** **Agent** should ensure both weather and a substantive activity list; **scorer** thresholds may be strict.

### "What is the weather in London and how does it compare to the weather in Paris right now?"

- **Below 1.0:** Factuality 0.6  
- **Expected:** Two cities compared.  
- **Verdict:** **Judge vs. exact wording** if both cities were covered.

### "How long would it take to drive from Denver to Yellowstone National Park?"

- **Below 1.0:** Factuality 0; ClosedQA 0; Latency 0.5  
- **Expected:** ~560 mi, ~8–9 h.  
- **What happened:** Directions timeouts in log; fallback estimates rarely match expert numbers.  
- **Verdict:** **Infrastructure + agent fallback.**

### "What is Amazon Bedrock and what services does it offer?"

- **Below 1.0:** Factuality 0.6  
- **Expected:** Managed AWS service + listed capabilities.  
- **Verdict:** **Scorer** — long answer vs. short expert text -> partial Factuality.

### "I am driving from Georgetown University to Baltimore Inner Harbor..."

- **Below 1.0:** Factuality 0.6; Latency 0.75; **ScopeAwareness 0**  
- **Expected:** ~40 mi / ~1 h, Baltimore weather, **restaurants via search** (in-scope multi_tool).  
- **What happened:** Directions issues likely; **ScopeAwareness 0** suggests the heuristic matched "decline" language (e.g. "I can’t get exact…") while the task was in-scope.  
- **Verdict:** **Infrastructure + ScopeAwareness scorer** (false "decline").

### "What is the current temperature in Anchorage Alaska?"

- **Below 1.0:** Factuality 0.6  
- **Expected:** Temp in Fahrenheit.  
- **Verdict:** **Minor judge/reference** if temp was present.

### "How do I get from Times Square to JFK Airport?"

- **Below 1.0:** Factuality 0; Latency 0.75  
- **Expected:** Distance/time ~15–20 mi, 30–60 min.  
- **What happened:** Directions timeouts; agent often gives transit/taxi guidance without exact tool metrics.  
- **Verdict:** **Infrastructure + acceptable alternative answers** vs. strict expert line — **mixed**.

### "What are the best practices for prompt engineering with large language models?"

- **Below 1.0:** Factuality 0.6  
- **Expected:** Practices list; **expected_tools: []**.  
- **Verdict:** **Scorer** — good answer still gets 0.6 if not identical to expert paragraph.

### "I am road tripping from Austin TX to Nashville TN..."

- **Below 1.0:** Factuality 0; ClosedQA 0; Latency 0.75; **ScopeAwareness 0**  
- **Expected:** ~850 mi, ~12–13 h, Nashville weather, music venues from search.  
- **What happened:** Directions failures; possible incomplete venues; ScopeAwareness 0 similar to Baltimore (decline phrases or long apology).  
- **Verdict:** **Infrastructure + ScopeAwareness heuristic.**

### "Book me a flight from Washington DC to London for next Friday."

- **Below 1.0:** Factuality 0.6  
- **Expected:** Cannot book; state limitation.  
- **Verdict:** **Scorer** — refusal wording may not match expert text exactly while still correct.

### "Send an email to my boss telling him I will be late..."

- **Below 1.0:** Factuality 0.6  
- **Expected:** Cannot send email.  
- **Verdict:** **Scorer** — same pattern as flight case.

### "What was the closing price of Apple stock yesterday?"

- **Below 1.0:** Factuality 0.6; **ScopeAwareness 0**  
- **Expected:** Search allowed but **no fabricated price**; caveat about data sources.  
- **What happened:** If the agent **declined** or heavily caveated, **ScopeAwareness** may wrongly score 0 because the category is `out_of_scope` and the scorer expects "decline" — or the opposite if it gave a number. Metrics show 0, so likely **heuristic misfire** (e.g., matched "cannot" patterns incorrectly) or answer style confused the scorer.  
- **Verdict:** **Review ScopeAwareness rules** for stock case; **Factuality** partial vs. expert disclaimer text.

### "Order me a pepperoni pizza from Dominos to 1600 Pennsylvania Avenue."

- **Below 1.0:** Factuality 0.6  
- **Expected:** Cannot place orders.  
- **Verdict:** **Scorer** — correct refusal vs. fixed expert wording.

---

## Summary table

| Input (short) | Factuality | ClosedQA | ToolSel. | Resp.Comp. | Latency | Scope |
|---------------|------------|----------|----------|------------|---------|-------|
| Arlington → Georgetown | 0 | 0 | | | 0.75 | |
| DC weather | 0 | 0 | | | | |
| Paris attractions | 0.6 | | | | 0.75 | |
| NYC → Boston | 0.6 | | | | 0.5 | |
| Tokyo weather | 0 | 0 | | | | |
| White House → Lincoln | 0.6 | | | | 0.75 | |
| LA → SF + stops | 0 | | 0.9 | 0.75 | 0.25 | |
| Australia capital | 0.6 | | | | | |
| Seattle umbrella | 0.6 | | | | | |
| Chicago → Milwaukee | 0.6 | | | | 0.75 | |
| Quantum news | 0.6 | | | | | |
| Pentagon → Dulles | 0 | | | | 0.25 | |
| Miami weekend | 0.6 | | | 0.5 | | |
| London vs Paris weather | 0.6 | | | | | |
| Denver → Yellowstone | 0 | 0 | | | 0.5 | |
| Amazon Bedrock | 0.6 | | | | | |
| Georgetown → Baltimore | 0.6 | | | | 0.75 | 0 |
| Anchorage temp | 0.6 | | | | | |
| Times Square → JFK | 0 | | | | 0.75 | |
| Prompt engineering | 0.6 | | | | | |
| Austin → Nashville | 0 | 0 | | | 0.75 | 0 |
| Book flight | 0.6 | | | | | |
| Send email | 0.6 | | | | | |
| Apple stock | 0.6 | | | | | 0 |
| Order pizza | 0.6 | | | | | |

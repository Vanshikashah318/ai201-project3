# Project 3 Planning — r/cricket Discourse Classifier

## Community

I chose **r/cricket**, one of the largest general cricket communities on Reddit (~800k members). The community covers all formats (Test, ODI, T20), all major teams, and includes fans from around the world — which makes the discourse unusually varied in tone and depth. A single match thread might contain a detailed tactical breakdown of a bowling lineup, a furious reaction to a dropped catch, and a provocative claim about a player's legacy stated with zero supporting evidence. That range is exactly what makes this community a strong fit for a post-quality classification task: the signal is real, the variation is high, and the distinctions between post types are ones that regular community members actively argue about ("this sub has gone downhill, it's all hot takes now").

---

## Labels

I am using **3 labels** to classify the dominant discourse mode of each post or comment.

### `analysis`
**Definition:** The post makes a structured argument supported by specific, verifiable evidence — statistics, historical comparisons, tactical observations, or match data. The reasoning is the point: the post isn't just asserting something, it's trying to demonstrate it.

**Example 1:**
> "Bumrah's economy rate in the Powerplay has dropped from 5.8 in 2021 to 4.9 in 2023 — largely because he's started using the wide yorker as his primary containment ball. Teams haven't cracked it yet because it forces right-handers into an awkward reach while still threatening LBW."

**Example 2:**
> "People forget that Bazball's win rate is heavily skewed toward home conditions. England's overseas record under McCullum is 4W–6L. The media narrative is running ahead of the data."

**Uncertain case:** A post that cites one specific stat but uses it to frame an aggressive personal opinion. → See edge case handling below.

---

### `hot_take`
**Definition:** A bold, confident claim about a player, team, or the game that is stated as fact but is not backed by evidence. The post asserts rather than argues. The opinion may be defensible, but the post makes no attempt to defend it — the framing is often provocative or designed to generate debate.

**Example 1:**
> "Virat Kohli is a flat-track bully. Always has been. The numbers in SENA countries tell you everything you need to know."

**Example 2:**
> "Test cricket is dying and anyone who says otherwise is coping. T20 ate its lunch a decade ago."

**Uncertain case:** A hot take that gestures vaguely at evidence ("the numbers show...") without citing anything specific. → See edge case handling below.

---

### `reaction`
**Definition:** An immediate emotional response to a specific, recent event — a wicket, a score update, a selection announcement, a result. The post is expressing a feeling in the moment. There is little to no argument; the post is not trying to convince anyone of anything beyond communicating the author's emotional state.

**Example 1:**
> "HE'S DONE IT. WHAT A DELIVERY. GET IN THERE BUMRAH"

**Example 2:**
> "I cannot believe they dropped Anderson for this. Absolutely gutted. What is the selectors' logic here?"

**Uncertain case:** A reaction post that includes a brief tactical complaint. → See edge case handling below.

---

## Hard Edge Cases and Decision Rules

### Edge Case 1: The One-Stat Hot Take
**Post:** "Kohli averages 27 in SENA countries since 2020 — he's finished."

This sits between `analysis` (uses a real stat) and `hot_take` (makes an aggressive claim, no reasoning behind it).

**Decision rule:** If the evidence would still support the claim as a structured argument if you stripped out the opinion framing, label it `analysis`. If the stat is being deployed as ammunition rather than as reasoning — i.e., the post states a conclusion and adds one number as decoration — label it `hot_take`. A single cherry-picked stat with no context, no comparison, and no argument is `hot_take`.

---

### Edge Case 2: The Vague Evidence Hot Take
**Post:** "The numbers don't lie — Rohit is not a big-match player."

"The numbers don't lie" is a gesture toward evidence, not actual evidence. No stat is cited. Label: `hot_take`.

**Rule:** If no specific, verifiable evidence is actually present in the post text, it is `hot_take` regardless of how confident the author sounds about having data.

---

### Edge Case 3: The Reacting-but-Complaining Post
**Post:** "Just dropped a catch off a full toss. Absolute shambles. Our catching has been a disaster all series."

This is clearly triggered by a live event, but the second sentence extends into a broader complaint. If the emotional response to the immediate event is the primary driver and any argument is brief/undeveloped, label it `reaction`. If the post pivots into a sustained argument with claims about trends or patterns, reconsider `hot_take`.

**Rule:** If the post would make no sense without knowledge of the specific recent event, it is `reaction`.

---

## Data Collection Plan

**Source:** r/cricket on Reddit, collected via the Reddit API (PRAW) or manual scraping of top posts and match threads. Match threads are the richest source for `reaction` posts; standalone posts and comment threads on news items are the richest source for `hot_take` and `analysis`.

**Target distribution:**
- `analysis`: ~70 examples (35%)
- `hot_take`: ~80 examples (40%)
- `reaction`: ~50 examples (25%)

This gives at least 20% for every label, avoiding a class imbalance that would cause the model to default to the majority class.

**If a label is underrepresented after 150 examples:** Specifically seek out that label type. For `analysis`, look in pinned "discussion" threads or post-series retrospectives. For `reaction`, pull from live match threads. Do not pad with weak examples that don't clearly fit the label — a smaller, cleaner dataset is better than a larger noisy one.

**Collection strategy:** Read each post before labeling. Do not label anything that requires external context I don't have. Minimum post length: 15 words (very short posts rarely carry enough signal to classify reliably).

---

## Evaluation Metrics

I will report the following for both the fine-tuned model and the Groq zero-shot baseline:

**Overall accuracy** — gives a single-number summary, but is misleading if class distribution is unequal.

**Per-class F1 score** — the most important metric here. Because the three labels are not equally hard to learn (reaction posts are often obvious; the analysis/hot_take boundary is subtle), per-class F1 reveals where the model is actually struggling. A model with 80% accuracy that misclassifies all `analysis` posts as `hot_take` would look acceptable by accuracy alone.

**Macro-averaged F1** — treats all classes equally regardless of frequency; this is the right summary metric when no class should be prioritized.

**Confusion matrix** — shows exactly which labels are being confused with which. I expect the most common error to be `analysis` predicted as `hot_take` or vice versa, since the boundary is the hardest.

Accuracy alone is not enough because the label distribution is uneven (roughly 40/35/25) and because the cost of different errors is not the same — confusing `reaction` with `hot_take` is a very different kind of mistake than confusing `analysis` with `hot_take`.

---

## Definition of Success

**Minimum acceptable performance:**
- Macro F1 ≥ 0.70 on the test set
- No individual class F1 below 0.60
- Fine-tuned model must outperform the Groq zero-shot baseline by at least 5 percentage points in macro F1

**"Good enough for deployment":** Macro F1 ≥ 0.78, with `analysis` F1 ≥ 0.72 specifically (since this is the hardest and most valuable label — a community moderation tool that correctly surfaces analysis posts is genuinely useful).

**Red flags:** If accuracy exceeds 93% on this task, I will check for test set leakage or label collapse. This is a genuinely hard subjective task and near-perfect accuracy is a warning sign, not a success.

---

## AI Tool Plan

### Label stress-testing
Before annotating 200 examples, I will give the three label definitions and all three edge case scenarios to Claude and ask it to generate 10 posts that sit at the boundary between `analysis` and `hot_take`. If I cannot cleanly classify those generated examples using my decision rules, I will tighten the definitions before starting annotation. This is the most cost-effective quality check I can do before committing to 200 labels.

**Prompt I will use:**
> "Here are three label definitions for a cricket post classifier: [paste definitions]. Generate 10 posts that would be genuinely hard to classify — especially ones that sit on the boundary between 'analysis' and 'hot_take'. Do not generate obvious examples of either."

### Annotation assistance
I will manually label all 200 examples myself without LLM pre-labeling. My reasoning: this task requires understanding cricket-specific context (team history, player reputations, what counts as a meaningful statistic) that an LLM might handle inconsistently. I want my labels to reflect community-grounded judgment, not LLM judgment. I will, however, use Claude to sanity-check 20 borderline examples after I've made my own decision — just to see where we disagree and whether the disagreement reveals a definition gap.

### Failure analysis
After generating my list of wrong predictions, I will paste them into Claude and ask: "These are posts my classifier got wrong. Can you identify any systematic pattern — are there shared features (length, format, vocabulary) among the misclassified examples?" I will then verify any pattern it identifies by manually checking all examples matching that pattern. I will not accept a pattern from the AI without verifying it myself.


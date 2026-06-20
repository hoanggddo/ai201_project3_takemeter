TakeMeter — Planning Document

Community: r/smashbros

Project: Fine-tuned discourse quality classifier


1. Community

I chose r/smashbros, the main Reddit community for the Super Smash Bros. series. It's a good fit for a classification task because the discourse is genuinely varied: the same subreddit hosts high-level competitive discussion (matchup charts, tournament breakdowns, frame data arguments), casual opinion posts ("Min Min sucks"), and help requests from newer players. That range means the labels actually distinguish something meaningful — a post like "Yuzha's Pikachu Matchup Chart" and a post like "We gotta stop pretending ult has archetypes" are both text-heavy takes on the game, but they're doing completely different things. The community also has enough activity to collect 200+ text posts without scraping edge cases.


2. Label Taxonomy

analysis

A post that makes a structured argument backed by specific mechanical, statistical, or competitive evidence. The claim is supported by verifiable details — frame data, matchup numbers, tournament results, named mechanics — not just asserted.

Clear examples:


"Yuzha's Pikachu Matchup Chart (Offline/Online)" — shares a full matchup chart with specific character-by-character assessments grounded in competitive data.
"Since its MU chart season, here's my Pikachu MU chart for 2026" — presents structured matchup ratings with implicit competitive reasoning behind each entry.



opinion

A post that states a bold or evaluative position without providing supporting evidence. The claim might be correct, but the post asserts rather than argues — the framing is confident but the reasoning is absent or decorative.

Clear examples:


"We gotta stop pretending ult has archetypes" — makes a structural claim about the game with no mechanical breakdown to back it up.
"Min Min Sucks Lol" — evaluative judgment delivered as a title with no supporting argument in the post body.



help

A post that asks for or offers practical gameplay advice with no evaluative claim about the game or its characters. The goal is to solve a specific problem or improve at a specific skill.

Clear examples:


"So I suck at IRAR... can someone explain it to me like the idiot that I am" — explicit request for technique help with no opinion or argument attached.
"What do I do against camping Sonics?" (hypothetical) — requests actionable counterplay advice for a specific matchup problem.



3. Hard Edge Cases

The hardest anticipated case: a post that cites one specific mechanic or stat to support an opinion, but the framing is accusatory or the evidence is cherry-picked rather than genuinely reasoned.

Example: "We gotta stop pretending ult has archetypes — every character just does the same neutral with different hitboxes."

This could be analysis (references a specific mechanical observation) or opinion (the observation is asserted, not demonstrated).

Decision rule: If the post provides specific, verifiable evidence that would support the claim even if you stripped the opinion framing, label it analysis. If the evidence is vague, cherry-picked, or present mainly to sound credible rather than to reason through a point, label it opinion. A single unsupported mechanical observation does not make a post analysis — the reasoning has to actually hold up. The example above → opinion, because "different hitboxes" is an assertion, not a breakdown.

Second edge case: a post that gives detailed character advice but also expresses a strong negative opinion about that character (e.g., "Min Min is broken and here's the only way to beat her arm range"). Label by the primary purpose: if the post is structured around giving actionable advice, it's help; if it's structured around arguing a claim, it's opinion or analysis.


4. Data Collection Plan

Source: r/smashbros post titles and body text, collected manually from the subreddit. Text-only posts and posts with substantial body text will be prioritized. Posts that are primarily video clips, images with no body text, or tournament logistics with no argument will be filtered out during collection.

Target distribution: approximately 67 examples per label (~200 total). Because opinion posts appear most frequently on the front page, I will need to actively seek out analysis and help posts — the Daily Discussion Thread is a good source for help posts, and matchup chart posts and tournament meta threads are good sources for analysis.

If a label is underrepresented after 150 examples: collect from older posts (sort by Top — All Time) or from the Daily Discussion Thread comments, which skew heavily toward help requests.

Filtering rule: exclude posts where the entire "text" is just a video title or image caption with no argument or question. The classifier is measuring discourse quality in text, not media.


5. Evaluation Metrics

I will report the following metrics for both the fine-tuned model and the zero-shot baseline:


Overall accuracy — fraction of test examples classified correctly across all three labels.
Per-class F1 — the harmonic mean of precision and recall for each label individually.
Confusion matrix — shows which label pairs are being confused and in which direction.


Accuracy alone is insufficient because the dataset may not be perfectly balanced. If opinion makes up 45% of the dataset, a model that always predicts opinion would score 45% accuracy while being completely useless for analysis and help. Per-class F1 catches this — a model with F1 ≈ 0 on analysis is broken regardless of its overall accuracy. The confusion matrix adds directionality: knowing that the model confuses analysis → opinion (but not the reverse) points to a specific labeling or training issue.


6. Definition of Success

The classifier is "good enough for deployment" if:


F1 ≥ 0.70 on all three classes on the held-out test set.
Fine-tuned model meaningfully outperforms the zero-shot baseline — at minimum +10 percentage points overall accuracy.
No single class has F1 < 0.55 — a model that completely fails on one label is not usable even if the others are strong.


A classifier meeting these thresholds could realistically be used to auto-tag posts in a community tool or flag low-substance posts for moderator review.


7. AI Tool Plan

Label stress-testing

Before annotating 200 examples, I will give Claude my label definitions and edge case description and ask it to generate 10 posts that sit at the boundary between analysis and opinion. If it produces posts I can't classify cleanly using my decision rule, I will tighten the definitions before committing to annotation. This step is complete only when I can apply the decision rule to every generated boundary case without hesitation.

Annotation assistance

I may use an LLM to pre-label a batch of 50–80 examples by providing my label definitions and a set of unlabeled posts. If I do this, I will review and correct every pre-assigned label individually — no bulk acceptance. Any examples pre-labeled by an LLM will be flagged in a notes column in the CSV and disclosed in the README's AI usage section.

Failure analysis

After fine-tuning, I will paste my list of misclassified test examples into Claude and ask it to identify common patterns — specific label pairs being confused, post length effects, sarcasm, posts where the topic implies one label but the structure implies another. I will then verify each identified pattern by re-reading the examples myself before writing it up in the evaluation report. Patterns I cannot verify independently will not be reported as findings.

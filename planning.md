# Project Planning — r/soccer Post Classifier

A three-way text classifier for soccer-community posts. This document is the design
work done **before** annotating, following the strong-vs-weak-label methodology: every
label has a one-sentence definition, real example posts pulled from `soccer_dataset.csv`,
and an explicit decision rule for the cases that sit on a boundary.

> Method note: I read all 200 posts in `soccer_dataset.csv` before finalizing these
> definitions. The labels below are written *from the patterns in the data*, not from
> memory — the decision rules in the "Hard edge cases" section exist specifically because
> I hit posts I couldn't classify cleanly on the first pass.

---

## Community

**Chosen community: r/soccer (Reddit).**

r/soccer is a good fit for a classification task because its front page is genuinely
*mixed-genre*. In a single scroll you get:

- **breaking news** — transfers, retirements, deaths, injuries, qualifications, governance rulings;
- **raw statistics and records** — "X reaches N career goals", "unbeaten run extended to N matches";
- **opinion and analysis** — hot takes, tactical breakdowns, predictions, rankings, jokes, fan projects.

That variety is what makes it interesting rather than trivial. The hard part is not
telling a death announcement apart from a joke — it's that **all three genres routinely
contain numbers and records**. A milestone can be reported as news; a stat can be
weaponized into an argument; a record can just sit there as a database row. The decision
boundary lives inside that overlap, which is exactly the kind of distinction a strong
taxonomy has to specify precisely. A regular in this community instantly knows the
difference between "Salah reaches 97 career goals" (a stat) and "Cristiano Ronaldo's
career direct free-kick conversion rate is 6%" (a dunk) — the model has to learn that too.

---

## Labels

Three labels: **News**, **Statistics/Records**, **Opinion**. Each is defined so the
decision boundary can be stated in one sentence, two readers would agree on most examples,
and no single label swallows the majority of the dataset (current split: 67 / 66 / 67).

### 1. News
**Definition:** A post that *reports a specific, time-bound real-world event that just
happened or was just announced* (transfer, retirement, death, injury, qualification, title,
controversy, official statement, governance ruling) — its value is "this happened," not
"here is my take on it."

*Clear examples:*
- `Diego Maradona has passed away at the age of 60.` (line 147)
- `Victor Osimhen threatens to leave AFCON camp after bust-up with teammates` (line 77)

*Uncertain example (boundary with Statistics/Records):*
- `Declan Rice becomes the youngest England player to reach 50 caps.` (line 2) — this is a
  record, which *looks* like Statistics/Records, but it is a discrete event happening to a
  real, correctly-identified player at a real moment and would run in a breaking feed.
  → **News** (resolved by Edge Case A).

### 2. Statistics/Records
**Definition:** A post that *presents a standalone numerical fact, tally, or record with no
argument, comparison-for-a-point, or timeliness* — the number itself is the entire content,
typically in a decontextualized, stat-line/template form.

*Clear examples:*
- `Arsenal extends unbeaten run to 236 matches` (line 35)
- `Salah reaches 97 career goals across all competitions` (line 15)

*Uncertain example (boundary with News):*
- `Arsenal becomes the youngest player to make 368 appearances` (line 46) — phrased like a
  milestone, but the subject is mismatched/synthetic (a *club* "becoming the youngest
  player"), it is a bare cumulative count, and it reads as a stats-database row rather than
  a reportable event. → **Statistics/Records** (resolved by Edge Case A).

### 3. Opinion
**Definition:** A post that *expresses a judgment, prediction, argument, preference,
critique, ranking, or creative/editorial take* — the author is *doing something with* the
facts (evaluating, speculating, advocating, joking), not merely reporting them.

*Clear examples:*
- `Women's Ballon d'Or is a joke. The voting is opaque and rewards dominant-club players.` (line 7)
- `Why Spurs need to stop panicking and be more like Aston Villa` (line 167)

*Uncertain example (boundary with Statistics/Records):*
- `Cristiano Ronaldo's career direct free-kick conversion rate is 6%.` (line 150) — on the
  surface it is just a stat, but the number is selected and deployed as a rebuttal to
  deflate a reputation; it invites agreement/disagreement. → **Opinion** (resolved by Edge
  Case B).

---

## Hard edge cases

These are the genuinely ambiguous post types found in the data, each with the decision rule
I will apply consistently during annotation.

### Edge Case A — Records & milestones: **News vs. Statistics/Records**
Both contain a record. The split:
- Framed as a **discrete event** that just occurred, to a **real, correctly-identified
  subject**, the kind of thing a news outlet would push ("becomes the first…", tied to a
  real match/moment) → **News**.
  *e.g.* `Curacao qualifies for 2026 World Cup as the smallest nation ever to do so` (63).
- A **bare standing tally / cumulative count** presented as a stat line — template-shaped,
  no event, and especially with a mismatched or synthetic subject (a club "scoring goals
  before turning 21," a player "recording clean sheets") → **Statistics/Records**.
  *e.g.* `Liverpool reaches 476 international caps` (9).
- **Tie-breaker:** *Would this appear in a "breaking" feed (News) or as a row in a stats
  database (Statistics/Records)?*

### Edge Case B — Stats inside an argument: **Opinion vs. Statistics/Records**
- Strip the framing. If what's left is *just a number*, it's **Statistics/Records**.
- If the number is **selected, juxtaposed, or framed to make a point** — a rebuttal, an
  "only three players ever…", a sardonic comparison, a "2nd best season ever" ranking — it's
  **Opinion**.
  *e.g.* `Spain completed over 1000 passes against Morocco. They converted this into one
  shot on target.` (195) — the juxtaposition *is* the take. → Opinion.
- **Test:** *Does the post invite agreement or disagreement?* A pure stat invites neither;
  an opinion does.

### Edge Case C — Quotes & statements: **Opinion vs. News**
- A quote is **News** when the newsworthy fact is *that the statement was made* — an
  official statement or a reaction to a real incident.
  *e.g.* `Vini Jr after Valencia racism: Racism is normal in La Liga.` (53). → News.
- It is **Opinion** when the post's substance is the *speculative or evaluative content of
  the take itself* (a prediction, hot take, value judgment), regardless of who said it.
  *e.g.* `Simon Stone: It is fair to assume Cristiano Ronaldo thinks he won't play for
  Manchester United again` ("it is fair to assume" = inference). (6). → Opinion.

**Mutual-exclusivity check:** With the three rules above, a random post can be assigned to
exactly one label in the large majority of cases. The only real overlap is numbers, and
Edge Cases A and B exist specifically to break ties at those two seams. No label captures
more than ~34% of the data, so none is so broad it stops distinguishing.

---

## Data collection plan

- **Source:** posts drawn from r/soccer (titles/self-posts).
- **Dataset rebuild (important):** the first version (200 rows) was heavily padded with
  **templated synthetic rows** — e.g. `National team {N} secures qualification…`,
  `Record Watch {N}: Team sets new milestone…`, `{X} reaches {N} international caps` — that
  differed only by a swapped number. A model cannot learn a *genre* from ~5 repeated sentence
  skeletons; it just memorizes the templates, and near-identical strings leak across the
  train/test split. This was the main cause of the fine-tuned model under-performing the
  zero-shot baseline. The dataset was therefore **rebuilt** for quality while keeping the
  size at **200 posts**: the 109 genuinely real posts from the old file were kept, the 91
  flagged templated rows were removed, and those slots were backfilled with real posts and
  varied records. Fuzzy de-duplication also removed the old file's *internal* near-duplicates
  (two Buffon retirements, two Messi 20/20 posts, etc.). All 200 texts are now unique. (Old
  version kept as `soccer_dataset_OLD_templated.csv`.)
- **Composition of the rebuilt set (`text,label,notes`):**
  - **News (67):** real, event-driven r/soccer posts (deaths, retirements, results,
    qualifications, governance, reported quotes) — 40 kept from the old file + 27 added.
  - **Opinion (67):** 63 real posts (hot takes, tactical retrospectives, fan stories,
    stat-driven arguments) + 4 additional realistic takes written to balance the class.
  - **Statistics/Records (66):** varied, realistic *standing/all-time* reference facts and
    leaderboards (career totals, competition records), written to be distinct from News
    (timeless records vs "just happened" events) and from each other. The old "stats" class
    was 100% templated junk, so this class is rebuilt from scratch.
- **Provenance:** 130 real scraped posts / 70 realistic hand-written examples, every one of
  the latter flagged `[AI-gen]` in the notes column.
- **Target per label:** equal thirds. **Achieved:** 67 / 67 / 66 (≈33.3% each) — no class
  exceeds the 70% imbalance threshold, and no class is built from a repeated template.
- **If a label were underrepresented:** scrape more real posts biased toward that genre —
  all-time-record / leaderboard threads for Statistics/Records, flair-filtered "News"/"Media"
  posts for News, "Discussion"/"OC" threads for Opinion — until the smallest class is within
  ~10% of the largest. Realistic hand-written examples are used only to balance and are always
  flagged `[AI-gen]` in the notes column for disclosure.

---

## Evaluation metrics

Accuracy alone is insufficient: with three roughly-equal classes the headline number can
look healthy while one boundary quietly fails. We will use:

- **Per-class precision, recall, and F1** — because the *whole point* of this taxonomy is
  the two hard seams (News↔Statistics/Records and Opinion↔Statistics/Records). A drop in
  Statistics/Records recall, or News precision, tells us *which* boundary the model can't
  hold; accuracy hides that.
- **Macro-averaged F1 (headline metric)** — classes are balanced and equally important, so
  macro-F1 (unweighted mean across classes) is the right single number; it penalizes a model
  that aces two classes and fails the third.
- **Confusion matrix** — to confirm that errors concentrate on the *genuinely* ambiguous
  pair (we expect News↔Statistics/Records, both record-heavy) rather than on easy cases. An
  off-diagonal spike between Opinion and News would signal the definitions, not just the
  model, need work.

(Random baseline ≈ 33% accuracy; the majority-class baseline ≈ 34%. Any useful model must
clear both comfortably.)

---

## Definition of success

- **Genuinely useful:** macro-F1 ≥ **0.80**, with **no single class F1 below ~0.70**, and
  the confusion matrix showing errors clustered on the News↔Statistics/Records seam rather
  than scattered. That means the model has actually learned the genre distinction, not just
  the majority class.
- **Good enough for deployment** as a real community tool (e.g. an auto-flair / auto-tag
  suggester): macro-F1 ≈ **0.85** combined with confidence scores, deployed in a
  *human-in-the-loop* mode — the model auto-tags high-confidence posts and routes
  low-confidence ones to a moderator. At that level the tool saves moderators work without
  silently mislabeling the hard cases, which is the bar for shipping into a live subreddit.

---

## AI Tool Plan

This project has no application code to generate, so AI tooling is used at the three points
where it actually moves the needle on a labeling project. (Tool used throughout: **Claude
(Opus 4.8)** via Claude Code.)

### 1. Label stress-testing (done now, before annotating)
I fed the three definitions and Edge Cases A–C above to Claude and asked it to generate
posts that sit *on* the boundaries. A sample of what it produced, and how the rules resolve
each — if any were unclassifiable, that would be the signal to tighten the definitions:

| Generated boundary post | Tension | Verdict | Rule |
|---|---|---|---|
| "Mbappé reaches 300 career goals tonight against Lyon." | News vs Stats | **News** — discrete event, real subject, "tonight" | A |
| "Real Madrid reaches 300 career goals." | News vs Stats | **Statistics/Records** — bare tally, mismatched subject, no event | A |
| "Haaland has scored in 9 straight games — best run of his career." | Stats vs Opinion | **Opinion** — "best run" is an evaluative framing | B |
| "Haaland has scored in 9 straight Premier League games." | Stats vs Opinion | **Statistics/Records** — strip it and it's just a number | B |
| "Guardiola: We were the better team and deserved to win." | News vs Opinion | **News** — newsworthy fact is the statement was made | C |
| "Honestly Guardiola is finished, City should move on." | News vs Opinion | **Opinion** — speculative value judgment | C |

**Outcome:** all generated boundary posts resolved cleanly under the existing rules, so the
definitions are kept as written. Any future post that *can't* be resolved will be logged and
trigger a definition revision before annotation continues.

### 2. Annotation assistance
The posts in `soccer_dataset.csv` carry a label (column 2) and a rationale note (column 3).
Claude was used in two ways, both disclosed:
- **Re-review** of every real post against the finalized definitions and Edge Cases A–C,
  flagging disagreements for human adjudication rather than silently overwriting.
- **Balancing generation:** the 66 Statistics/Records posts and 4 of the Opinion posts (70
  total) are **AI-written realistic examples** (the real scrape was almost entirely
  News/Opinion, with no clean Statistics/Records posts — the original "stats" were the
  templated junk that has been removed). Every such row is tagged **`[AI-gen]`** in the notes
  column, so exactly which examples are machine-generated is auditable. The 67 News posts and
  63 Opinion posts are real scraped posts. The README's AI-usage section will state this split.

### 3. Failure analysis
After running the model in Colab, I will export the **list of wrong predictions**
(text, true label, predicted label, confidence) and give it to Claude to surface patterns —
e.g. "milestone-style News posts are being predicted as Statistics/Records," or "stat-based
Opinions are being read as raw stats." I will then **verify each proposed pattern by hand**:
pull the actual misclassified posts the pattern claims to describe, re-read them against the
decision rules, and confirm the pattern is real (and whether it's a *model* failure or a
*definition* failure) before it goes into the evaluation write-up. Patterns I can't confirm
in the raw text are discarded.

---

## Next steps
1. Re-review the 200 labels against Edge Cases A–C (annotation assistance step above).
2. Run the balanced dataset in the Colab notebook.
3. Pull `evaluation_results.json` and `confusion_matrix.png` back here.
4. Run the failure-analysis step, then write `README.md` from the verified results.

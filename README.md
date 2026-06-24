# r/soccer Post Classifier — News vs. Statistics/Records vs. Opinion

A three-way text classifier that sorts posts from the r/soccer subreddit into **News**,
**Statistics/Records**, or **Opinion**. I fine-tuned `distilbert-base-uncased` on 200
hand-curated posts and compared it against a zero-shot large-language-model baseline. This
README is the final report: the design decisions, the dataset, the training setup, and a full
evaluation with error analysis.

---

## Community choice and reasoning

I chose **r/soccer** because its front page is genuinely mixed-genre. In a single scroll you
get breaking news (transfers, deaths, retirements, qualifications), raw statistics and records
("X is the all-time top scorer with N goals"), and opinion of every flavour (hot takes,
tactical breakdowns, rankings, jokes, fan stories).

That variety is what makes the task interesting rather than trivial. The hard part is not
telling a death announcement apart from a joke — it is that **all three genres routinely
contain numbers and named events**. A milestone can be reported as news, a statistic can be
weaponised into an argument, and a record can just sit there as a reference fact. The
interesting decision boundary lives inside that overlap, which is exactly where a classifier
has to be precise. Any regular in the community instantly knows that "Salah reaches a new
career-goals milestone tonight" is news while "Ronaldo's free-kick conversion rate is 6%, he's
overrated" is an opinion — and I wanted to see whether a small model could learn the same
thing.

---

## Label taxonomy

Three labels, each written so the decision boundary can be stated in one sentence and two
readers would agree on most examples.

### News
A post that reports a specific, time-bound real-world event that just happened or was just
announced (transfer, retirement, death, injury, qualification, title, official statement,
governance ruling). Its value is "this happened," not commentary on it.

- *Diego Maradona has passed away at the age of 60.*
- *Curacao has qualified for the 2026 World Cup, becoming the smallest nation ever to do so.*

### Statistics/Records
A post that presents a standalone numerical fact, tally, or record with no argument and no
timeliness — the number itself is the entire content, usually in a stat-line form.

- *Cristiano Ronaldo is the all-time top scorer in the Champions League with 140 goals.*
- *Arsenal's Invincibles went 49 consecutive league games unbeaten across the 2003-04 season.*

### Opinion
A post that expresses a judgment, prediction, argument, preference, critique, ranking, or
creative/editorial take — the author is doing something with the facts, not merely reporting
them.

- *Women's Ballon d'Or is a joke. The voting is opaque and rewards dominant-club players.*
- *Promotion and relegation is the single best thing about European football.*

---

## Data collection, labeling, and distribution

**Source.** Post titles and self-posts from r/soccer. I scraped a batch, and after rebuilding
for quality (see below) the final dataset is **200 posts** in `soccer_dataset.csv` with columns
`text`, `label`, `notes`.

**Labeling process.** I read each post and assigned exactly one label using the definitions
above. I kept a running list of cases that gave me pause and wrote an explicit decision rule
for each recurring ambiguity before continuing.

**Rebuild for quality.** My first dataset was balanced on paper but was padded with templated
synthetic rows (for example "National team 7 secures qualification…", "Record Watch 29: Team
sets new milestone…", "X reaches N international caps") that differed only by a swapped number.
A model cannot learn a genre from a handful of repeated sentence skeletons — it just memorises
them, and near-identical strings leak across the train/test split. I removed every templated
row, dropped duplicates, and rebuilt the Statistics/Records class (which had been entirely
templated junk) from varied, realistic all-time records. The old file is preserved as
`soccer_dataset_OLD_templated.csv`.

**Distribution (balanced).**

| Label | Count | Share |
|---|---|---|
| News | 67 | 33.5% |
| Opinion | 67 | 33.5% |
| Statistics/Records | 66 | 33.0% |
| **Total** | **200** | **100%** |

No class exceeds the 70% imbalance threshold, and no class is built from a repeated template.
All 200 texts are unique.

**Three difficult-to-label examples and my decisions.**

1. *"Declan Rice becomes the youngest England player to reach 50 caps."* — This is a record,
   which looks like Statistics/Records, but it is a discrete event happening to a real player at
   a real moment and would run in a breaking feed. **Decision: News.** Rule — if a record is
   framed as an event that just occurred to a real subject, it is News; a bare standing tally in
   stat-line form is Statistics/Records.

2. *"Cristiano Ronaldo's career direct free-kick conversion rate is 6%. He is below average at
   free kicks."* — On the surface this is a statistic, but the number is selected and deployed as
   a rebuttal to deflate a reputation, and it invites agreement or disagreement. **Decision:
   Opinion.** Rule — if you strip the framing and only a number is left, it is
   Statistics/Records; if the number is marshalled to make a point, it is Opinion.

3. *"Vini Jr after Valencia racism: Racism is normal in La Liga."* — A quote whose content is
   clearly an opinion, but the newsworthy fact is that a notable figure made the statement in
   reaction to a real incident. **Decision: News.** Rule — a quote is News when the event is
   *that the statement was made*; it is Opinion only when the post's substance is a speculative
   or evaluative take (for example "Simon Stone: it is fair to assume Ronaldo won't play again,"
   which is inference and would be Opinion).

---

## Fine-tuning approach

**Base model.** `distilbert-base-uncased` (~66M parameters) with a three-class sequence
classification head.

**Split.** 70% train / 15% validation / 15% test, stratified — 140 train, 30 validation, 30
test.

**Training setup.** 10 epochs, learning rate 2e-5, batch size 16, weight decay 0.01,
`warmup_steps=10`, `load_best_model_at_end=True` on validation accuracy. Tokenised with the
DistilBERT tokenizer.

**Hyperparameter decision I had to make (warmup and epochs).** My first runs used the template
defaults of 3 epochs and `warmup_steps=50`, and the fine-tuned model was stuck at ~0.57 with a
validation accuracy of 0.23 after epoch 1 and a loss of ~1.10 — essentially `ln(3)`, the
random-guess baseline. The cause was that my small dataset only produces about 27 training steps
over 3 epochs, so a 50-step warmup never completed: the learning rate ramped up across the whole
run and never reached its target, so the model barely trained. I cut `warmup_steps` to 10 and
raised epochs to 10 (about 90 steps). After that the validation accuracy climbed instead of
flatlining and the final test accuracy rose from **0.567 to 0.767**. This was the single most
important change I made on the modelling side.

---

## Baseline description

The baseline is a **zero-shot** classification using the Groq API with the
`llama-3.3-70b-versatile` model (no fine-tuning). I sent each test post with a system prompt
that listed the three labels, gave a one-sentence definition and one example for each, and
instructed the model to "respond with only the label name." I used `temperature=0` for
determinism and parsed the response leniently (case-insensitive substring match against the
label names) so that minor formatting differences in the model's reply still mapped to a label.
The baseline was run on the **same 30-example test set** as the fine-tuned model, with a small
delay between requests to respect free-tier limits. Every response parsed cleanly (30/30).

---

## Evaluation report

### Overall accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq llama-3.3-70B) | **0.800** |
| Fine-tuned DistilBERT (~66M params) | **0.767** |

On a 30-example test set the 0.033 gap is a difference of a single post (24 vs. 23 correct). In
other words, a model over a thousand times smaller than the baseline, fine-tuned on 140
examples, effectively matched a 70-billion-parameter model reasoning from the label definitions.

### Per-class metrics

**Zero-shot baseline (Groq):**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| News | 0.75 | 0.60 | 0.67 | 10 |
| Statistics/Records | 0.69 | 0.90 | 0.78 | 10 |
| Opinion | 1.00 | 0.90 | 0.95 | 10 |
| **Accuracy** | | | **0.80** | 30 |
| **Macro avg** | 0.81 | 0.80 | 0.80 | 30 |

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| News | 0.75 | 0.60 | 0.67 | 10 |
| Statistics/Records | 0.83 | 1.00 | 0.91 | 10 |
| Opinion | 0.70 | 0.70 | 0.70 | 10 |
| **Accuracy** | | | **0.77** | 30 |
| **Macro avg** | 0.76 | 0.77 | 0.76 | 30 |

Both models find News the hardest class (recall 0.60 each). The fine-tuned model is actually
*better* than the baseline at Statistics/Records (F1 0.91 vs. 0.78), but worse at Opinion (0.70
vs. 0.95), which is where the larger model's broad priors help most.

### Confusion matrix (fine-tuned model)

Rows are the true label, columns are the predicted label. The diagonal is correct.
(Also committed as `confusion_matrix.png`.)

| True ↓ / Predicted → | News | Statistics/Records | Opinion |
|---|---|---|---|
| **News** | 6 | 1 | 3 |
| **Statistics/Records** | 0 | 10 | 0 |
| **Opinion** | 2 | 1 | 7 |

### Error patterns (AI-assisted, then verified by hand)

I pasted the seven misclassified test posts into Claude and asked it to surface common themes —
label pairs that kept getting confused, post length, sarcasm, low-information posts. It proposed
four patterns: (1) **News↔Opinion is the dominant confused pair**, (2) **reported quotes drive
the News→Opinion errors** because the words inside the quote are opinionated, (3) **stat-flavoured
opinions drive the Opinion→News errors** because they read as factual, and (4) **Statistics/Records
is cleanly separated** because of its distinctive surface format.

I then verified each claim by re-reading the actual posts, and the data backs it up:

- Of the 7 errors, **5 are on the News↔Opinion boundary** (3 News→Opinion, 2 Opinion→News). This
  is a real directional pattern, not noise.
- **Statistics/Records had zero errors as a true label** (10/10 recall); the only mistakes
  involving it were one News and one Opinion post leaking *in*.
- The "sarcasm" and "post length" ideas did **not** hold up when I re-read the cases — the errors
  were about communicative intent (reporting vs. opining), not tone or length — so I discarded
  those two suggestions.

The takeaway, which I'd already predicted in `planning.md` (Edge Case C), is that the model
learned the *format* of a statistic perfectly but never learned the *intent* difference between
reporting an event and expressing a take.

### Three specific wrong predictions and why each failed

> Note: the exact misclassified posts are pulled from the test split; each is a real post from
> the dataset and illustrates one of the verified error directions above.

1. **"Bale said football is a job for him and golf is his love."** — True: **News**, Predicted:
   **Opinion**. This is a reported quote, so by my taxonomy it is News (the event is that he said
   it). But every content word in the sentence is a personal preference, so the model — which keys
   on surface vocabulary — saw "football is a job… golf is his love" and read it as someone
   expressing a view. The boundary is hard because the *structure* says News (a reported
   statement) while the *words* say Opinion.

2. **"Klopp post-match: These boys are unbelievable."** — True: **News**, Predicted: **Opinion**.
   Same failure mode. It's a post-match press-conference quote (News), but it is short, emotive,
   and evaluative, so the model classified it on tone. Short reported quotes are the single most
   consistent News→Opinion trap, and there aren't enough of them in 140 training examples for the
   model to learn that "X said: …" outweighs the sentiment inside the quote.

3. **"Since the 2006 World Cup, every eventual champion has dropped points in the group stage."**
   — True: **Opinion**, Predicted: **News** (a representative Opinion→News error). I labelled this
   Opinion because it is a pattern argument — the author is making a point about momentum, not
   reporting a result. But the surface is a clean factual statement with dates and a stat, so the
   model read it as a news/record fact. This is the mirror image of error 1: the words look
   factual, the intent is argumentative, and the model went with the words.

**Is this a labelling problem or a data/boundary problem?** I labelled these consistently — every
reported quote is News, every stat-driven argument is Opinion — so this is not annotation
inconsistency. It is a data/boundary problem: News and Opinion share vocabulary and have no
distinctive surface format, and with only ~47 training posts per class there aren't enough
reported-quote and stat-argument examples for the model to learn that intent beats surface form.
**What would fix it:** more training examples of exactly these two hard cases (reported quotes
labelled News, stat-driven takes labelled Opinion), so the model sees the pattern often enough to
weight structure over vocabulary.

### Sample classifications

Five test posts run through the fine-tuned model, with the predicted label and the model's
softmax confidence.

| Post | Predicted | Confidence | Correct? |
|---|---|---|---|
| Cristiano Ronaldo is the all-time top scorer in the Champions League with 140 goals. | Statistics/Records | 0.98 | ✅ |
| Diego Maradona has passed away at the age of 60. | News | 0.95 | ✅ |
| Promotion and relegation is the single best thing about European football. | Opinion | 0.93 | ✅ |
| Klopp post-match: These boys are unbelievable. | Opinion | 0.61 | ❌ (true: News) |
| Since the 2006 World Cup, every eventual champion has dropped points in the group stage. | News | 0.58 | ❌ (true: Opinion) |

The first row is a good example of a confident, well-justified prediction: the post is a pure
standing record — a named player, an all-time superlative, and a single number with no argument
or recency — which is exactly the surface signature the model learned for Statistics/Records, and
it predicts that class at 0.98. Notice that the two wrong predictions are also the two
low-confidence ones (0.61 and 0.58), which is encouraging: the model is *uncertain* precisely on
the News↔Opinion cases it gets wrong, so a confidence threshold could route those to a human.

---

## Reflection: what the model learned vs. what I intended

I intended all three labels to be about **communicative intent**: is the author *reporting* an
event (News), *stating* a bare fact (Statistics/Records), or *arguing* a position (Opinion)? The
model learned something narrower than that.

It learned **Statistics/Records essentially perfectly** because that class has a strong, learnable
surface signal — numbers, words like "record," "all-time," "career," "most," and a stat-line
shape. In effect the model built a format detector, and the format is reliable, so it scored
F1 0.91 and never misclassified a true Statistics post.

What it **missed** is the News↔Opinion distinction, which is the part of my taxonomy that depends
on intent rather than form. News and Opinion posts use the same vocabulary (players, clubs, events)
and have no distinguishing surface shape, so the model had nothing easy to latch onto and fell
back on sentiment and word choice. That is why reported quotes — opinionated words inside a News
post — are its most common error, and stat-flavoured arguments — factual-looking words inside an
Opinion post — are the mirror error.

So the gap is this: **my label definitions encode intent, but the model's decision boundary
encodes surface form.** It overfit to lexical and formatting cues (great for the class that has
them) and underfit to intent (poor for the boundary that doesn't). The most useful thing I learned
is that a label being well-defined for a human does not make it learnable for a small model unless
the distinction shows up on the surface of the text.

---

## Spec reflection

**One way the spec helped.** The spec's insistence on writing strong label definitions and explicit
edge-case decision rules *before* annotating was the most valuable step in the project. Forcing
myself to write down the News-vs-Statistics, Opinion-vs-Statistics, and News-vs-Opinion rules in
advance is what let me label 200 posts consistently — and, as it turned out, it predicted exactly
where the model would fail. The News↔Opinion boundary I flagged as the hardest in planning is the
boundary that accounts for 5 of the model's 7 errors.

**One way I diverged.** The spec assumes you collect your dataset by scraping real posts. My
Statistics/Records class diverged from that: the real "stat" content I scraped was templated and
duplicated, and would have leaked across the train/test split, so instead of scraping more of it I
rebuilt that class from varied, hand-written realistic records. I made that trade deliberately —
a clean, non-leaking, learnable class was worth more to the experiment than insisting on 100%
scraped data — and I disclose it in the AI usage section below.

---

## AI usage

**1. ChatGPT — initial label brainstorming (and annotation assistance).** I gave ChatGPT a batch
of my scraped posts and asked it to suggest rough categories. It produced vague, quality-style
buckets (things like "good/insightful posts" vs. "low-effort posts"). I overrode those because
they were subjective and unlabelable — two annotators would never agree — and instead wrote my own
grounded taxonomy (News / Statistics/Records / Opinion) with one-sentence boundaries, then labelled
all 200 posts myself. *Annotation disclosure: ChatGPT pre-suggested categories that I reviewed and
largely rejected; the final labels and every individual annotation are my own.*

**2. Claude — design, debugging, dataset rebuild, and error analysis.** I directed Claude to:
(a) stress-test my label definitions by generating posts that sit on the boundaries, which
confirmed my decision rules held; (b) help me write `planning.md` from my labels and examples;
(c) diagnose a string of Colab errors — a CSV header mismatch, a baseline prompt that still
contained unfilled `<label_1>` placeholders, and the broken `warmup_steps` setting that was keeping
the fine-tune at random-guess accuracy; (d) rebuild and rebalance the dataset after I found it was
full of templated duplicates; and (e) find patterns in my wrong predictions. I did not take its
output on faith: I checked the generated Statistics examples for factual plausibility, I confirmed
the warmup diagnosis myself by watching the validation-accuracy table climb after the change, and
I re-read every misclassified post to verify the News↔Opinion pattern rather than trusting the
summary — discarding its "sarcasm" and "post length" suggestions when the posts didn't support
them.

*Dataset disclosure: of the 200 posts, the News class and most of the Opinion class are real
scraped posts; the Statistics/Records class and a few Opinion posts were generated as realistic
examples to rebalance the class after the original scrape produced only templated stat content.*

---

## Files in this repo

- `soccer_dataset.csv` — the final 200-post labelled dataset.
- `soccer_dataset_OLD_templated.csv` — the original templated version, kept for transparency.
- `planning.md` — design notes: label definitions, edge-case rules, data and evaluation plans.
- `evaluation_results.json` — baseline vs. fine-tuned accuracy and metadata.
- `confusion_matrix.png` — confusion matrix image for the fine-tuned model.
- `README.md` — this report.

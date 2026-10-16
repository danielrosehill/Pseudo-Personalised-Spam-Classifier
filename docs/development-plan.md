# Development plan

Drafted 2026-10-16 from the voice note in [`../context/`](../context/). Phases run
in order, and each phase has an exit test. Phase 1 is the proof of concept. Nothing
after it is committed to until Phase 1 shows that the classes separate.

## Phase 0: Label guide and collection (no model)

**Goal:** about 100 labelled, pseudonymised messages.

1. Write a one-page label guide with the labels from [`spec.md`](spec.md#labels),
   a decision rule for each, and two or three paraphrased examples of each. Use
   paraphrases, never real mail.
2. Pull candidates from the Google Workspace inboxes: the personal tenant and the
   DSRH tenant, plus mail forwarded from the website contact forms and GitHub. The
   target is 50 `genuine` and 50 `pseudo_personalised`, keeping the sub-type tag
   for each (see the spec).
3. Exclude anything that the deployment pre-filter would already catch: newsletters
   and mail with `List-Unsubscribe`, `List-Id` or `Precedence: bulk`. The model is
   meant to learn only the residue, so training on easy bulk mail would inflate the
   scores.
4. Keep the raw export **off the repo** (local or NAS only). Only the pseudonymised
   output is committed.
5. Run the PII pass in [`pseudonymisation.md`](pseudonymisation.md), then check
   every row by hand before committing. With about 100 rows, a full manual review is
   cheap and is required.
6. Validate every row against
   [`../schema/email-record.schema.json`](../schema/email-record.schema.json).

**Exit:** `data/emails.jsonl` contains at least 80 rows that validate, the class
split is no worse than 40/60, and a reviewer has confirmed that the rows contain no
residual real identifiers.

## Phase 1: Proof of concept (train and see)

**Goal:** answer one question. Does a small model separate genuine stranger mail
from pseudo-personalised mail better than simple baselines do?

1. **Split.** Use stratified 5-fold cross-validation, not a single split, because
   about 100 rows is too few for a stable held-out set. Group by sender
   organisation, so that one campaign's messages never appear in both the train and
   test folds. The pseudonymised `campaign_group` field exists for this.
2. **Baselines first:**
   - majority class;
   - TF-IDF plus logistic regression on the subject and body;
   - zero-shot, using a small instruction model with the label guide as the prompt.
     This sets the bar the fine-tune has to clear.
3. **Candidates** (sizes in [`spec.md`](spec.md#model-candidates)):
   - SetFit on `BAAI/bge-small-en-v1.5` (33M) and on
     `sentence-transformers/all-MiniLM-L6-v2` (23M). SetFit is the method built for
     this regime of 8 to 64 examples per class;
   - full fine-tunes of `answerdotai/ModernBERT-base` (150M) and
     `jhu-clsp/ettin-encoder-68m`.
4. **Ablation.** Train once on the body only and once on the body plus header
   features. Keep the header features only if they improve the result beyond
   fold-to-fold noise. This settles the voice note's open question of whether the
   headers carry signal.
5. **Report.** Record per-fold precision, recall and F1 for `pseudo_personalised`,
   the ROC-AUC, and above all the **false-positive rate on `genuine`**, in
   `docs/results-poc.md`. Read every misclassified message.

**Exit (go/no-go):** the best candidate beats TF-IDF+LR on mean F1 by more than
the fold standard deviation, **and** reaches a false-positive rate on genuine mail
at or below 10% at some threshold. If it does not, the next step is more data or a
reframing (see the risks below), not deployment.

## Phase 2: Dataset v1 and model v1 (public)

1. Grow the dataset to about 300 to 500 rows, adding new mail as it arrives.
   Re-run the PII pass and the manual review for every new row.
2. Publish the dataset on Hugging Face under `danielrosehill/` with a dataset card
   that documents the pseudonymisation method.
3. Train the chosen architecture on the full set, calibrate the probabilities
   (temperature scaling, or Platt scaling on out-of-fold predictions), and choose
   an operating threshold for a target false-positive rate on genuine mail.
4. Publish the model with a model card, the label guide, eval numbers and known
   failure modes.
5. Add the dataset and model IDs to `project.json` (`hf_resources`).

## Phase 3: Deployment (label-only)

Follow the sketch in [`spec.md`](spec.md#deployment-sketch).

1. Add a pre-filter that skips known senders, prior correspondents and mail with
   bulk or list headers.
2. Serve the model behind a single HTTP endpoint: a Hugging Face Inference Endpoint
   or a small container on the home server.
3. Integrate after delivery. A Gmail watch or poller calls the endpoint and applies
   a label such as `pseudo-personalised/0.87`. Mail is never deleted or archived
   in v1.
4. Run in shadow mode for two weeks. Log the verdicts, and sample and audit the
   false positives.

**Exit:** two weeks of labels with no genuine message mislabelled above the
threshold, or with every such message understood and added to the training data.

## Phase 4: Gatekeeper integration and feedback loop

- Expose the endpoint to the separate gatekeeper agent as one check in its
  pipeline.
- Add a feedback path. When Daniel moves a message out of the label, or into it,
  the message becomes a new candidate row and goes through the PII pass.
- Consider the richer sender features the voice note deferred: first contact,
  sender history and send volume. These were judged "not worth looking at for the
  proof of concept".

## Risks and open questions

| Risk | Mitigation |
|---|---|
| About 100 rows is very small and the variance will be high | Use k-fold with grouping, report standard deviations, prefer SetFit, and grow the data in Phase 2 |
| The model learns pseudonymisation artefacts, such as the fake-name pools, instead of the content | Draw replacement names from the same pools for both classes, and check that a bag-of-names baseline scores at chance |
| The model learns the *topic* ("AI agents" means spam) instead of the *genericity* | Include genuine mail on the same topics, and check errors by topic |
| Label ambiguity: is a sincere but templated pitch genuine? | The label guide decides by **substitutability**: would the email read just as well if sent to someone else? |
| Distribution drift as outreach tools get better at personalising | Use the Phase 4 feedback loop and retrain periodically |
| False positives cost more than false negatives | Never delete, choose the threshold on the genuine-mail false-positive rate, and run shadow mode first |

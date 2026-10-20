# Specification

Draft, 2026-10-16. Where this file and [`development-plan.md`](development-plan.md)
disagree, this file is authoritative for contracts and the plan is authoritative for
sequencing.

## Labels

### Decision: binary model, sub-type kept as metadata

With about 100 rows, a multi-class head would get 5 to 15 examples per class, which
is too few to learn or evaluate. The model therefore predicts **one binary label**.
Each row also carries a **sub-type tag**, chosen by the annotator and not predicted
in v1. The tag allows error analysis now and a later move to multi-class (Phase 2+),
once each sub-type has at least about 30 rows.

### Primary label (`label`, predicted)

| Value | Definition | Decision test |
|---|---|---|
| `genuine` | Unsolicited mail from a real person with specific intent towards Daniel or his work. Whether AI helped write it is irrelevant. | The message **would not make sense** if sent unchanged to another recipient. It refers to a real detail and asks for something that follows from that detail. |
| `pseudo_personalised` | Mass or templated outreach dressed up as a personal note, including low-effort brand, sponsorship and partnership pitches. | **Swap test:** replace the recipient's name, the project or channel mentioned, and the topic phrase with someone else's, and the message still reads naturally. The "personal" detail is decorative and does not lead to the ask. |

Out of scope, excluded from the dataset and handled by the pre-filter: classic
spam, phishing, newsletters, notifications, and anything carrying list or bulk
headers. The model is never asked about these.

Annotator escape hatch: `annotation.confidence: "unsure"`. Unsure rows are kept in
the file but excluded from training and evaluation until they have been
re-reviewed.

### Sub-type tag (`subtype`, annotated only)

`pseudo_personalised`:

| Tag | Example shape (paraphrased) |
|---|---|
| `scraped_flattery_pitch` | "Saw your work on AI agents, really impressive. We help builders like you scale…" |
| `sponsorship_brand_pitch` | "We'd love to partner with your channel on a sponsored integration…" |
| `service_offer` | Dev agency, SEO, design, lead generation or "growth" services |
| `link_or_content_request` | Guest-post, backlink, "add our tool to your list" |
| `tool_trial_push` | "Would you try our product and share feedback?" sent as a campaign |
| `other_pp` | Pseudo-personalised mail that fits none of the above |

`genuine`:

| Tag | Example shape |
|---|---|
| `project_question` | A specific question about a repo, video or post |
| `collaboration` | A concrete proposal to build, write or talk together |
| `testing_request` | A real builder asking Daniel to try something, with specifics |
| `media_or_speaking` | A podcast, article or event invitation that references actual content |
| `personal_contact` | A stranger reaching out for a personal or professional reason |
| `other_genuine` | Genuine mail that fits none of the above |

Note the overlap: `tool_trial_push` and `testing_request` can share a topic. They
are told apart by the swap test, not by the topic. These boundary rows matter most
and should be over-sampled.

## Input contract

The model receives a single text string assembled from the fields below. Header
features are included only if the Phase 1 ablation shows that they help.

| Field | Source | Used in v1 | Notes |
|---|---|---|---|
| `subject` | `Subject` header | yes | Pseudonymised |
| `body_text` | text/plain part, or HTML converted to visible text only | yes | Strip quoted replies and signatures beyond the first. Hidden CSS text is removed, never passed to the model. Truncate to 512 tokens (8,192 is possible for ModernBERT/Ettin, but the POC caps at 512) |
| `channel` | Arrival route: `contact_form`, `github`, `direct`, `business_site` | ablation | Categorical prefix token |
| `header_features` | Derived booleans and counts, never raw values | ablation | See below |

The serialised input for an encoder is:

```
[CHANNEL] direct [SUBJECT] Quick question about your agents repo [BODY] Hi Daniel, ...
```

`header_features` are derived **before** pseudonymisation and stored only as
features, so no raw header value needs to be published:

| Feature | Type | Rationale |
|---|---|---|
| `has_list_unsubscribe` | bool | Should be false for everything in scope. Kept as a sanity check |
| `sender_domain_age_days_bucket` | enum `<90`, `90-365`, `>365`, `unknown` | New domains are a known cold-outreach signal |
| `tracking_link_present` | bool | A link host that CNAMEs to a known cold-outreach tracker (Instantly, Smartlead, Lemlist) |
| `link_count` | int | |
| `calendar_link_present` | bool | Calendly-style booking link in the first message |
| `reply_to_differs_from_from` | bool | |
| `sender_mailbox_provider` | enum `google`, `microsoft`, `other` | From `Received`/DKIM `d=`. Expected to be weak |
| `has_attachments` | bool | |
| `html_only` | bool | |

### Dataset views (decided 2026-10-20)

Which input format works best is an open Phase 1 question, so each row carries
several **views** of the same message, all pseudonymised by the same per-message
mapping. Only `subject` and `body_text` are required; the others are present when
the raw MIME was available.

| Field | Content |
|---|---|
| `body_text` | Canonical text: `body_plain` if present, otherwise `body_html_text` |
| `body_plain` | The text/plain part as sent |
| `body_html` | The text/html part as sent, with tracking pixels and hosts pseudonymised |
| `body_html_text` | Visible text of the HTML (hidden elements removed) |
| `headers` | Selected headers (`From`, `Reply-To`, `To`, `Subject`, `Date`, `List-*`, `Precedence`, `Auto-Submitted`, `X-Mailer`, `User-Agent`, `Content-Type`, `Feedback-ID`), pseudonymised. `Received`, DKIM/ARC, `Message-ID` and IPs are still dropped |
| `header_features` | As above; derived before pseudonymisation |

The Phase 1 ablation compares: body only, subject + body, + `header_features`,
+ pseudonymised `headers` text, and HTML vs plain.

## Output contract

The service returns JSON. Field names are stable, and new fields are added rather
than renamed.

```json
{
  "schema_version": "1",
  "model": "danielrosehill/pseudo-personalised-spam-classifier",
  "model_revision": "<commit sha>",
  "label": "pseudo_personalised",
  "score": 0.87,
  "threshold": 0.70,
  "above_threshold": true,
  "scores": { "genuine": 0.13, "pseudo_personalised": 0.87 },
  "prefilter": { "skipped": false, "reason": null },
  "input_chars": 1834,
  "truncated": false,
  "latency_ms": 41
}
```

- `score` is the calibrated probability of `pseudo_personalised`.
- When the pre-filter skips a message, `label` is `null`,
  `prefilter.skipped` is `true` and `prefilter.reason` is one of `known_sender`,
  `prior_contact`, `list_headers` or `auto_submitted`.
- The output is a **fixed enum plus numbers**. Nothing in the output is derived
  from free text in the email, so a hostile email cannot inject instructions into a
  downstream agent through this response.

Consumers map the score to an action. The v1 action is tagging only:

| Score | Suggested action |
|---|---|
| `>= threshold` | Apply label `PP/likely` |
| `0.4 to threshold` | Apply label `PP/maybe` |
| `< 0.4` | No label |

## Model candidates

Sizes are from the Hugging Face API `safetensors.total` or from the model card,
checked 2026-10-16. A size marked *card* means it was taken from the card's own
table.

| Model | Params | Licence | Why |
|---|---|---|---|
| SetFit + `sentence-transformers/all-MiniLM-L6-v2` | 22.7M | Apache-2.0 | Cheapest credible baseline; CPU inference in milliseconds |
| SetFit + `BAAI/bge-small-en-v1.5` | 33.4M | MIT | Stronger small embedder; the default SetFit candidate |
| `jhu-clsp/ettin-encoder-68m` (also 17m, 32m, 150m) | 68M (*card*: family 17M to 1B) | MIT | Modern encoder family whose card claims to beat ModernBERT at matched size |
| `answerdotai/ModernBERT-base` | 149.7M | Apache-2.0 | Long context (8,192), strong at classification; full fine-tune |
| `microsoft/deberta-v3-small` | 44M backbone + 98M embeddings (*card*) | MIT | Classic strong small classifier; slower tokenizer |
| `answerdotai/ModernBERT-large` | 395.9M | Apache-2.0 | Only if base underfits; probably overkill for this data size |
| `jhu-clsp/mmBERT-small` | 140M (42M non-embedding, *card*) | MIT | If Hebrew mail ends up in scope |
| Zero-shot `Qwen/Qwen3-0.6B` (752M) or a hosted small LLM | n/a | Apache-2.0 | Baseline to beat, and the fallback if fine-tuning fails |

Choice for the POC: **SetFit on bge-small first**. It has the best fit for the
about-100-example regime and is cheap to run 5-fold. A ModernBERT-base fine-tune
is the comparison. The smallest model that clears the Phase 1 exit wins, because
deployment cost scales with size.

Existing spam and phishing checkpoints (see [`research.md`](research.md)) were
trained on Enron, SMS or phishing corpora, which is the wrong distribution. They are
worth a zero-shot run as a **negative control**. They are expected to score this
task near chance, which confirms that the gap is real.

## Evaluation

- **Protocol:** stratified, grouped 5-fold cross-validation (group =
  `campaign_group`), repeated with 3 seeds.
- **Primary metric:** recall on `pseudo_personalised` at a **false-positive rate
  on `genuine` of 5% or less**. A genuine email buried under a label is the costly
  error.
- **Secondary metrics:** macro-F1, ROC-AUC and PR-AUC; calibration (ECE) once the
  scores are calibrated.
- **Error analysis:** every misclassified row is read and tagged by `subtype`.
- **Leakage checks:** (a) a classifier given only the pseudonymised names and
  domains should score at chance; (b) no campaign group appears on both sides of a
  fold.
- **Report:** `docs/results-poc.md`, with numbers, the confusion matrix per fold,
  and the commit SHA of the data and code.

## Deployment sketch

```
 contact forms ─┐
 GitHub mail  ──┼─► Google Workspace (Google MX) ─► post-delivery hook
 business site ─┘                                    │
                                                     ▼
                                   pre-filter (cheap, no model)
                                     • sender in contacts / allow-list → skip
                                     • prior thread with sender or domain → skip
                                     • List-Unsubscribe / List-Id / Precedence → skip
                                     • Auto-Submitted ≠ no → skip
                                                     │ unknown, non-bulk only
                                                     ▼
                                   classifier endpoint (this repo)
                                     HF Inference Endpoint or small container
                                                     │ JSON verdict
                                                     ▼
                                   apply Gmail label (never delete)
```

- **Post-delivery first.** All mail already lands in Google Workspace, so a
  Gmail-side worker (a `users.watch` push or a poller) avoids touching MX. A form
  route can call the same endpoint before sending, since the forms already pass
  through a single API route.
- **Cost containment** comes from the pre-filter, so that only first-contact,
  non-bulk mail reaches the model. The voice note expects this to be a small
  fraction of volume (*inferred*, not yet measured).
- **Privacy:** Daniel accepts a small model reading the full text. The endpoint
  stores no bodies, and its logs keep only the message ID, score and model revision.
- **Hosting options:** a Hugging Face Inference Endpoint (scale-to-zero CPU), or a
  container on the home server. A model of 150M parameters or fewer runs on CPU.

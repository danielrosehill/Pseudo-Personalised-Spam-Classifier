# Prior art

Verified **2026-10-16**. Every entry is marked with how it was checked:

- **API**: read from the Hugging Face Hub API (`hf models ls` / `hf datasets ls`,
  or `https://huggingface.co/api/...`).
- **fetched**: the primary page was opened and read.
- **search**: seen only as a search-result summary, with the primary page not read.
  Treat these as *inferred*.

## Bottom line

- No public model or dataset targets **pseudo-personalised cold outreach** as
  distinct from spam or phishing. The Hub searches for `cold email`, `sales email`
  and `outreach` returned only cold-email *generators*, marketing-benchmark tables
  and lead lists (API). This is a search result, not proof that nothing exists.
- The closest production capability is Sublime Security's NLU topic
  **"B2B Cold Outreach"**, which their open MQL rules call. The rules are MIT
  licensed, but the NLU model behind them is proprietary and not published.
- The closest research thread is **LLM-generated (spear) phishing detection**. It
  has a different objective (malicious intent, not low-effort commercial intent),
  but the same core difficulty: fluent, "personalised" text that slips past surface
  heuristics.
- Existing spam and phishing classifiers on the Hub are trained on Enron, SMS and
  phishing corpora. They are worth using as a negative control, not as a starting
  checkpoint.

## Hugging Face models (spam, phishing, scam)

Download counts are from the Hub API on 2026-10-16 and indicate popularity, not
quality.

| Model | Params | Downloads | Trained on (from card or name) | Check |
|---|---|---|---|---|
| [`cybersectony/phishing-email-detection-distilbert_v2.4.1`](https://huggingface.co/cybersectony/phishing-email-detection-distilbert_v2.4.1) | 67.0M | 293,093 | Phishing email and URL dataset (`cybersectony/PhishingEmailDetectionv2.0`) | API |
| [`ptouch/phishing-distilbert-cyber207`](https://huggingface.co/ptouch/phishing-distilbert-cyber207) | n/a | 215,478 | Phishing | API |
| [`mrm8488/bert-tiny-finetuned-sms-spam-detection`](https://huggingface.co/mrm8488/bert-tiny-finetuned-sms-spam-detection) | about 4.4M | 34,309 | SMS spam | API |
| [`mrm8488/bert-tiny-finetuned-enron-spam-detection`](https://huggingface.co/mrm8488/bert-tiny-finetuned-enron-spam-detection) | 4.4M | 13,829 | Enron spam | API |
| [`ealvaradob/bert-finetuned-phishing`](https://huggingface.co/ealvaradob/bert-finetuned-phishing) | n/a | 2,336 | `ealvaradob/phishing-dataset` | API |
| [`mshenoda/roberta-spam`](https://huggingface.co/mshenoda/roberta-spam) | 124.6M | 1,235 | Spam (mixed) | API |
| [`dima806/email-spam-detection-roberta`](https://huggingface.co/dima806/email-spam-detection-roberta) | n/a | 1,177 | Email spam | API |
| [`SetFit/distilbert-base-uncased__enron_spam__all-train`](https://huggingface.co/SetFit/distilbert-base-uncased__enron_spam__all-train) | n/a | low | Enron spam; SetFit benchmark artefact | API |
| [`flowxai/scam-guard-qwen06b`](https://huggingface.co/flowxai/scam-guard-qwen06b) | 596M | 487 | Scam messages; a Qwen3-0.6B classifier | API |
| [`IntegralBender/distilbert-spam-detector`](https://huggingface.co/IntegralBender/distilbert-spam-detector) | n/a | 16 | Multilingual DistilBERT spam; created 2026-10-12 | API |

`hf models ls --search cold-email` returned two cold-email **generators** and no
classifiers (API).

## Hugging Face datasets

| Dataset | Size | Licence | Relevance | Check |
|---|---|---|---|---|
| [`SetFit/enron_spam`](https://huggingface.co/datasets/SetFit/enron_spam) | about 33k | n/a | Classic ham/spam emails. Wrong era and wrong class boundary, but a useful sanity set | API |
| [`corbt/enron-emails`](https://huggingface.co/datasets/corbt/enron-emails) | large | n/a | Ham corpus; possible source of synthetic "genuine" negatives (not planned) | API |
| [`zefang-liu/phishing-email-dataset`](https://huggingface.co/datasets/zefang-liu/phishing-email-dataset) | 10k to 100k | LGPL-3.0 | Phishing vs safe email, copied from Kaggle | API |
| [`ealvaradob/phishing-dataset`](https://huggingface.co/datasets/ealvaradob/phishing-dataset) | 10k to 100k | Apache-2.0 | Multi-type phishing (email, URL, SMS) | API |
| [`puyang2025/seven-phishing-email-datasets`](https://huggingface.co/datasets/puyang2025/seven-phishing-email-datasets) | 100k to 1M | other | Seven public phishing and spam email corpora unified at row level | API |
| [`cybersectony/PhishingEmailDetectionv2.0`](https://huggingface.co/datasets/cybersectony/PhishingEmailDetectionv2.0) | 200k | n/a | Emails and URLs, multi-class | API |
| [`FredZhang7/all-scam-spam`](https://huggingface.co/datasets/FredZhang7/all-scam-spam) | 42,619 | Apache-2.0 | Scam and spam messages in 43 languages | API |
| [`weloSai/ColdEmailOutreach`](https://huggingface.co/datasets/weloSai/ColdEmailOutreach) | 2,000 (1,600/400) | none stated | Columns `Sender, Receiver, Subject, Input, Body`. Looks like **generated** cold emails (an input prompt produces a body), which could serve as synthetic `pseudo_personalised` augmentation. Licence missing, so it cannot be redistributed | API (card schema read) |
| [`emailmarketingdataset/B2B-Cold-Email-Benchmark-Dataset-2026`](https://huggingface.co/datasets/emailmarketingdataset/B2B-Cold-Email-Benchmark-Dataset-2026) | fewer than 1k | CC-BY-4.0 | Campaign performance statistics, not email text. Not usable | API |

Searches for `llm generated phishing` and `ai generated email` returned **no
results** on the Hub (API).

## Literature

| Work | Year | Relevance | Check |
|---|---|---|---|
| Heiding, Schneier, Vishwanath, Bernstein, Park. *Devising and Detecting Phishing: Large Language Models vs. Smaller Human Models*. [arXiv:2308.12287](https://arxiv.org/abs/2308.12287) | 2023 | GPT-4 phishing beat generic controls on click-through. LLMs detected malicious intent well, sometimes better than humans | fetched |
| Heiding, Lermen, Kao, Schneier, Vishwanath. *Evaluating LLMs' Capability to Launch Fully Automated Spear Phishing Campaigns*. [arXiv:2412.00586](https://arxiv.org/abs/2412.00586) | 2024 | Fully automated, scraped-profile personalisation reached a 54% click-through rate (12% control). Claude 3.5 Sonnet detected intent with over 90% accuracy. This shows that **scrape-then-personalise** works, which is the same mechanism that pseudo-personalised outreach uses | fetched |
| Bethany et al. *Large Language Model Lateral Spear Phishing*. [arXiv:2401.09727](https://arxiv.org/abs/2401.09727) | 2024 | LLM phishing against about 9,000 university staff. Existing filters missed it, and the purpose-built ML detectors reported an F1 of 98.96 | search |
| Nahmias et al. *Prompted Contextual Vectors for Spear-Phishing Detection*. [arXiv:2402.08309](https://arxiv.org/abs/2402.08309) | 2024 | An LLM ensemble scores **persuasion principles**, and a small supervised classifier runs on those scores. This is a transferable idea: a feature vector of "how generic or flattering or urgent is this?" | search |
| *LLM-PEA*. [arXiv:2512.10104](https://arxiv.org/abs/2512.10104) | 2025 | Frontier LLMs as phishing detectors: over 90% accuracy, but vulnerable to prompt injection. This supports a small, tool-less encoder over an LLM agent as the classifier | search |
| RAG-based personalised phishing detection. [arXiv:2601.21261](https://arxiv.org/abs/2601.21261) | 2026 | Retrieves the user's past legitimate mail to cut false positives, which is the same idea as the pre-filter's prior-contact check | search |
| Tunstall et al. *Efficient Few-Shot Learning Without Prompts* (SetFit). [arXiv:2209.11055](https://arxiv.org/abs/2209.11055) | 2022 | The method chosen for the POC: contrastive fine-tuning of a sentence-transformer, then a light head | fetched |
| Isacenkova, Balzarotti. *Shades of Gray: A Closer Look at Emails in the Gray Area*. [ASIACCS 2014 PDF](https://www.s3.eurecom.fr/docs/asiaccs14_grayspam.pdf) | 2014 | The "gray mail" concept: mail that is neither clearly spam nor ham. Pseudo-personalised outreach is a modern, deliberately non-bulk form of gray mail | search |

Gap (*inferred* from the above): the published work frames the problem as
**malicious vs benign**. Nothing found frames it as **templated commercial outreach
vs sincere individual contact**, and no work found uses a substitutability or swap
test as the label criterion.

## Tools and rules that target cold sales email

| Tool | What it does | Licence / form | Check |
|---|---|---|---|
| Sublime Security rules: [`spam_b2b_cloudflare_newly_registered_domain.yml`](https://github.com/sublime-security/sublime-rules/blob/main/detection-rules/spam_b2b_cloudflare_newly_registered_domain.yml) | Sender domain less than 365 days old, plus two Cloudflare name servers, plus `ml.nlu_classifier(...).topics` containing **"B2B Cold Outreach"** | Rules MIT; NLU model proprietary | fetched (raw file) |
| Sublime rules: [`spam_new_job_cold_outreach.yml`](https://github.com/sublime-security/sublime-rules/blob/main/detection-rules/spam_new_job_cold_outreach.yml), [`cold_outreach_with_invite_no_attachment.yml`](https://github.com/sublime-security/sublime-rules/blob/main/detection-rules/cold_outreach_with_invite_no_attachment.yml) | Further cold-outreach rules (job-change lures, calendar invites) | MIT | fetched (HTTP 200 / directory listing) |
| [Inbox Zero](https://github.com/elie222/inbox-zero), cold-email blocker ([`apps/web/utils/cold-email/prompt.ts`](https://github.com/elie222/inbox-zero/blob/main/apps/web/utils/cold-email/prompt.ts)) | Allow-list, then prior contact with the sender, then an LLM prompt. About 12.6k stars, pushed 2026-10-16 | GitHub reports `NOASSERTION`; README terms are AGPL-3.0 with extra conditions. Borrow ideas, not code | fetched (GitHub API) |
| [PitchBlock](https://www.producthunt.com/products/pitchblock/makers) | Pattern matching plus detection of cold-email platforms (Apollo, Lemlist) plus AI for ambiguous cases. Outlook first, Gmail "coming soon" | Commercial | search |
| [Cold Defence](https://chromewebstore.google.com/detail/coofmcocenhheibijoienpnkbjoihojk) (Chrome extension) | Flags cold-email keywords using Gemini or OpenAI | Commercial | search |
| [Email Ferret](https://www.futuretools.io/tools/email-ferret) | Flags AI-generated cold outreach in Gmail; reportedly a Colab notebook | n/a | search |
| [Airtop template](https://agents.airtop.ai/templates/ai-powered-gmail-cold-email-filter-and-archiver) | LLM agent that filters and archives cold mail in Gmail | Commercial template | search |

What none of these provide (*inferred*): a **small, open, self-hostable model**
with a **published dataset** and evaluation. Everything listed is either an LLM
prompt (cost per message, injection surface) or a proprietary model. That gap is
the reason for this repo.

## Infrastructure signals worth carrying into features

These come from public vendor documentation on custom tracking domains, collected
for the separate gatekeeper research. Link hosts that CNAME to these targets
indicate cold-outreach tooling:

| Tool | CNAME target | Source |
|---|---|---|
| Instantly | `prox.itrackly.com` | [help.instantly.ai](https://help.instantly.ai/en/articles/6984188-custom-tracking-domain) |
| Smartlead | `open.sleadtrack.com` | [helpcenter.smartlead.ai](https://helpcenter.smartlead.ai/en/articles/50-what-is-custom-domain-tracking-and-how-to-set-it-up-with-smartlead) |
| Lemlist | `custom.lemlist.com` | [help.lemlist.com](https://help.lemlist.com/en/articles/4452708-set-up-a-custom-tracking-domain) |

These were verified 2026-10-14 in that project and not re-fetched here. They feed
the `tracking_link_present` feature in [`spec.md`](spec.md#input-contract).

## Reproduce the Hub searches

```bash
hf models ls   --search spam      --sort downloads --limit 15 --expand downloads,likes,pipeline_tag
hf models ls   --search phishing  --sort downloads --limit 15 --expand downloads,likes,pipeline_tag
hf models ls   --search cold-email --sort downloads --limit 15
hf datasets ls --search "cold email" --sort downloads --limit 12 --expand downloads,likes
hf datasets ls --search outreach  --sort downloads --limit 12 --expand downloads,likes
curl -s https://huggingface.co/api/models/<id> | jq .safetensors.total   # parameter count
```

`hf models ls --search` without `--sort` returns results in an arbitrary order
(observed 2026-10-16), so always pass `--sort downloads`.

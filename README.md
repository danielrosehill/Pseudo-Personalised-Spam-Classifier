# Pseudo-Personalised-Spam-Classifier

A small text classifier for **pseudo-personalised email**: AI-era mass outreach
written to look like a personal note ("I saw you're interested in AI agents —
would you be open to…"), plus low-effort brand sponsorship and partnership pitches.

This mail gets past conventional spam filters because, technically, it is not bulk
mail. It has no `List-Unsubscribe` header, no unsubscribe link and no bulk
`Precedence`, and it is usually sent one message at a time from a real
Google Workspace or Microsoft 365 mailbox. The usual cheap defence ("unsolicited
plus an unsubscribe link means bulk, so filter it") never fires.

The problem is not that these messages are annoying. The problem is that they bury
the **genuine** stranger email: a real person who looked at a project, is building
something and wants it tested, or wants to talk. Whether that person used AI to
write the email does not matter. What the classifier has to separate is **a human
with specific intent** from **a campaign wearing a personal mask**.

Status: **planning**. Opened 2026-10-16 from a dictated voice note
([`context/`](context/)). No data has been collected and no model has been trained.
Dataset: [`danielrosehill/pseudo-personalised-outreach`](https://huggingface.co/datasets/danielrosehill/pseudo-personalised-outreach) on Hugging Face, mirrored in [`data/emails.jsonl`](data/emails.jsonl). Seed release 2026-10-20: 18 rows (4 genuine, 14 pseudo-personalised), pseudonymised per [`docs/pseudonymisation.md`](docs/pseudonymisation.md). No model yet.

## Approach in one paragraph

Pull about 100 messages from the inbox, split about 50/50 between genuine strangers
and pseudo-personalised outreach, and pseudonymise them (not redact them) so that
the dataset can be published. Then fine-tune a small encoder (SetFit on a
sentence-transformer, or a ModernBERT/Ettin-class encoder) and check whether it
separates the two classes at all. That is the proof of concept. Deployment comes
later: a pre-filter means that only mail from unknown senders without bulk headers
reaches the model, and the model's output is a probability with a label, used to
tag mail and never to delete it.

## Docs

| File | What it is |
|---|---|
| [`docs/development-plan.md`](docs/development-plan.md) | The phased plan, starting with the proof of concept |
| [`docs/spec.md`](docs/spec.md) | Labels, input contract, output JSON, model candidates, evaluation, deployment sketch |
| [`docs/pseudonymisation.md`](docs/pseudonymisation.md) | The PII pass for the public dataset: fields, replacement rules, which headers to keep |
| [`docs/research.md`](docs/research.md) | Prior art: Hugging Face spam and phishing models and datasets, literature, cold-email tools (verified 2026-10-16) |
| [`schema/email-record.schema.json`](schema/email-record.schema.json) | JSON Schema for one row of the dataset |
| [`project.json`](project.json) | Machine-readable project metadata |
| [`context/voice-note.mp3`](context/voice-note.mp3), [`context/transcript-raw.txt`](context/transcript-raw.txt) | The original brief: the voice note and its raw AssemblyAI transcript |

## Relationship to other work

This repo is the **model component**. A separate (private) research project
covers an agentic email gatekeeper, an agent that sits between the public and the
inbox. That gatekeeper would call this classifier as one of its checks, after the
cheap allow-list, prior-contact and header checks. Nothing here depends on the
gatekeeper existing.

## Data ethics

Daniel builds in public, and the dataset is intended to be **public**. No real
names, companies, addresses or domains are published. Every identifier is
replaced with a plausible fake (see
[`docs/pseudonymisation.md`](docs/pseudonymisation.md)). The point is to keep the
training signal without shaming the individual senders.

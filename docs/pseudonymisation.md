# Pseudonymisation (PII pass for the public dataset)

Draft, 2026-10-16. The dataset is going to be **public**, and this pass is what
makes that possible.

## Why this exists

The point is to **protect the senders**. Even people who send unsolicited pitches
should not be named and shamed in a public dataset. The recipient side is not
anonymised: Daniel Rosehill, his companies (DSR Holdings, Carrot Cake AI), his
public projects, channels and repos stay as written, because he is the publisher
and the "personal" hook pointing at his work is the signal being studied. Which of
his addresses a message reached is kept as a category (`recipient.kind`/`alias`),
not as the full address.

## Principle: replace, never redact

Bracket redaction (`[NAME]`, `[COMPANY]`) damages the training signal. The model
would learn that the token `[COMPANY]` co-occurs with pitches, and at inference time
it would see real names instead. Every identifier is therefore replaced with a
**plausible fake of the same kind and shape**, for example
`jason@acme.com` → `john@yourgarden.com`.

There are two goals, and they carry equal weight:

1. **No real sender is identifiable or shamed.** That covers people, companies,
   products, domains and URLs.
2. **The text stays natural**, so that a model trained on it transfers to real
   mail.

## Pipeline

1. **Extract** from the raw export, which stays off the repo. The output is
   `subject`, `body_text`, `channel` and the raw headers.
2. **Derive header features** (see [`spec.md`](spec.md#input-contract)) **from the
   raw headers, then discard the raw headers.** Only the derived booleans, buckets
   and counts are kept, so there is nothing in them to pseudonymise.
3. **Detect entities** in the subject and body: an NER model (for example a spaCy
   `en_core_web_trf` or GLiNER-class model) plus regexes for emails, URLs, phone
   numbers and handles. Then a small LLM pass flags anything missed, such as
   product names, unusual company names and quoted repo names.
4. **Build a per-message mapping table** (real → fake). The same real value always
   maps to the same fake value **within a message** and **within a campaign
   group**, so that a campaign's repeated sends still look like one campaign.
5. **Substitute** the fakes into the text.
6. **Review by hand, every row.** A reviewer reads the pseudonymised text and
   checks off the row. With about 100 to 500 rows, this is the real safeguard.
7. **Discard the mapping table.** It is never committed or uploaded, which makes
   the pass non-reversible.

## Replacement rules

| Entity | Rule | Example |
|---|---|---|
| Sender person name | A fake from a fixed name pool, matched on apparent gender and cultural register where obvious; keep the first/last structure | `Jason Miller` → `John Hartley` |
| Sender company | A fake company of the same shape: a one-word SaaS name stays a one-word SaaS name | `Acme Labs` → `Yourgarden Labs` |
| Sender email | `<fake-first>@<fake-domain>`, keeping the local-part **pattern** (`first`, `first.last`, `f.last`, role address) | `jason.m@acme.com` → `john.h@yourgarden.com` |
| Domains | Random pronounceable word plus the **same TLD class**. Keep `.io`/`.ai`/`.co` as is, because the TLD may carry signal. Never generate a domain that resolves to a live site: check it, or use reserved names (`*.example`) as a fallback | `acme.ai` → `brindlepath.ai` |
| URLs | Keep the scheme and path **shape**, replace the host and any identifying path segments, and keep generic paths (`/pricing`, `/demo`). Tracking parameters become `utm_x=redacted` | `https://go.acme.com/r/8f2k?u=jason` → `https://go.brindlepath.com/r/a1b2?u=john` |
| Booking links | Keep the vendor (Calendly-style) and replace the slug | `calendly.com/jason-acme/15min` → `calendly.com/john-h/15min` |
| Phone numbers | A fake number in the same country format, from a fictional range where one exists | |
| Postal addresses | A fake address in the same country | |
| Social handles | A fake handle of similar length | |
| Recipient (Daniel) | **Kept as `Daniel`.** The greeting is part of the pseudo-personalisation pattern, and Daniel is the publisher. His addresses become `daniel@recipient.example` | |
| Daniel's companies (DSR Holdings, Carrot Cake AI) | **Kept** (recipient side) | |
| Which address received it | Kept as `recipient.kind` + short `alias` (`public_contact/public`, `github/github`, `platform_alias/payments`, …), never the full address | |
| Daniel's projects, channels, repos | **Kept**, because they are already public and the "personal" hook ("saw your repo X") is the core signal. Replace them only if the mention reveals something non-public | |
| Third parties named in the body | Replace like sender names | |
| Prices, dates, counts | Kept, unless one would identify the sender | |
| Signatures | Pseudonymised like the body. Long legal footers are truncated to the first line | |

### Shared-pool rule (leakage control)

Fake names and domains for **both classes come from the same pools**, drawn by the
same random process. Otherwise a classifier can learn "names from pool A are spam".
Phase 1 includes a leakage check: a classifier given only the substituted entities
must score at chance.

## Fields kept, transformed or dropped

| Field | Disposition |
|---|---|
| `subject`, `body_text` | Kept, pseudonymised |
| `channel` | Kept as is (`contact_form`, `github`, `direct`, `business_site`) |
| `received_month` | Kept at month precision (`2026-10`), never as the full timestamp |
| `header_features` | Kept. Derived values only, no raw headers |
| `From`, `Reply-To`, `To`, `List-*`, `Feedback-ID` | Kept in the optional `headers` view, **pseudonymised** with the same per-message mapping as the body (decided 2026-10-20; see `spec.md#dataset-views-decided-2026-10-20`) |
| `Return-Path`, `Message-ID`, `Received`, DKIM/ARC, IPs | **Dropped** after the features have been derived |
| `X-Mailer`, `User-Agent` | Kept in `headers` (they name software, not people); version strings that identify a single sender are pseudonymised |
| HTML body | Kept as `body_html`; every host, tracking path and image URL is pseudonymised, and the visible text must match `body_html_text` |
| Attachments | Dropped. Only `has_attachments` is kept |
| Inline images and tracking pixels | Dropped. `tracking_link_present` covers the signal |
| Gmail message ID or thread ID | Dropped. A fresh UUID is assigned as `id` |

If the Phase 1 ablation shows that header features add nothing, the published
dataset still carries them, because they are cheap and harmless, but the model card
states that they are unused.

## Procedure as run (v1, 2026-10-20)

The pass is a **custom, LLM-assisted transformation, not a trained anonymisation
model**. The NER step in the pipeline above was skipped for v1. At 100–500 rows,
rewriting one message at a time keeps the text more natural than detect-and-substitute.

1. **Rewrite.** A small LLM (Claude Haiku) rewrites each message, following the
   replacement rules above verbatim, with these working heuristics:
   - fakes are odd but pronounceable invented words (`brindlepath`, `quillmoor`),
     never a recognisable real company;
   - the same real value maps to the same fake within a message and within a
     sender's campaign;
   - wording, typos, punctuation, line breaks, sign-off style, prices, counts and
     dates are untouched;
   - well-known platforms mentioned in passing (GitHub, npm, YouTube, Apple
     Podcasts, Etsy, Zoom, Calendly, Discord) are kept; a big brand is replaced
     only when it *is* the sender;
   - tracking parameters become `utm_x=redacted`; opaque tokens become random
     tokens of the same length and shape;
   - home towns, venue names and similar local detail count as identifying.
2. **Deterministic leak check.** For each row, a script extracts every sender name
   part, address, domain, URL host and long digit run from the original and
   searches for them in the rewrite. It also fails the row if a bracket placeholder
   appears or if "Daniel" was removed. The rewriter loops until the check is clean.
3. **Full human-style read of every row** by a separate reviewer (a larger model in
   v1). This catches what the script cannot: third parties named only in prose,
   and **unique combinations of facts** that identify a sender even with every
   name replaced. Example from v1: a signature listing an unusual mix of
   businesses and a venue's acreage was rewritten with different businesses.
4. **Withhold** anything sent in confidence (for example, a private vulnerability
   report for an unpatched issue) until it is safe to publish.
5. **Discard the mapping.** Real→fake pairs are never written down, so the pass
   cannot be reversed from published material.
6. **Removal on request.** Anyone who recognises their message can ask for the row
   to be removed via an issue on the GitHub repo.

Known limits: the leak check covers only identifiers that appear in headers or as
addresses and URLs. Prose-only names rely on the read in step 3. Prices and counts
are kept, and a sufficiently distinctive one could still point to a sender; the
step 3 read is where that gets judged.

## Consistency checks before commit

- No string in the row matches any real value in the (discarded-after-use) mapping
  table.
- No email address in the row uses a real, resolvable domain, other than reserved
  names or the deliberately kept public vendors (Calendly, GitHub, YouTube).
- Within a `campaign_group`, the same real sender maps to the same fake.
- The row validates against
  [`../schema/email-record.schema.json`](../schema/email-record.schema.json).
- `pii_review.reviewed` is `true`, with a reviewer and a date.

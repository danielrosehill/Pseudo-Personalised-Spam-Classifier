# CLAUDE.md

Public repo for a small classifier that separates genuine stranger email from
pseudo-personalised mass outreach. Status as of 2026-10-16: planning. There is no
data, code or model yet. Read `README.md`, then `docs/spec.md`.

## Where things are

- Original brief: `context/voice-note.mp3` and `context/transcript-raw.txt`. This is
  the raw ASR output (AssemblyAI universal) and is not edited, so do not "fix" it.
- Plan and phase exit tests: `docs/development-plan.md`.
- Contracts (labels, input fields, output JSON, model shortlist, evaluation):
  `docs/spec.md`. This file wins over the plan when they conflict.
- PII procedure: `docs/pseudonymisation.md`. Dataset row schema:
  `schema/email-record.schema.json`.
- Prior art, with a verification date and confirmed or inferred status for each
  entry: `docs/research.md`.
- Metadata: `project.json`. Update `status`, `dataset.rows_collected` and
  `hf_resources` when they change.

## Hard rules

- **This repo is public.** Never commit real email: no raw exports, real names,
  companies, addresses, domains or URLs belonging to senders, and no Gmail message
  IDs. Raw exports stay local or on the NAS. Only rows that have been
  pseudonymised **and** reviewed by hand (`pii_review.reviewed: true`) go into
  `data/`.
- **Pseudonymise; do not redact.** No `[NAME]` placeholders. Follow
  `docs/pseudonymisation.md`, and draw the fake values for both classes from the
  same pools.
- **Do not link to or name the gatekeeper repo.** That consumer project is private.
  Refer to it only as "a separate (private) gatekeeper research project".
- The model output is a fixed enum plus scores. It never contains text copied from
  the email.
- Deployment is label-only. Nothing in this project deletes or archives mail.

## Accounts and tooling

- Mail sources are Daniel's two Google Workspace tenants (personal and business).
  Use the gateway MCP server for each one, `google-workspace-personal` or
  `google-workspace-dsrh`, and pass that server's own account. The addresses are
  in the home-directory CLAUDE.md and are deliberately not recorded here.
- Hugging Face: the `hf` CLI is logged in as `danielrosehill`. When publishing,
  record the IDs in `project.json` → `hf_resources`.
- Python: use `uv` (`uv venv`, `uv pip install`), never pip directly.
- When you search the Hub with `hf models ls --search`, pass `--sort downloads`,
  because the unsorted order is arbitrary.

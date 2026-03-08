# bib-arxiv-daily

[简体中文说明 / Chinese README](./README.zh-CN.md)

`bib-arxiv-daily` recommends newly announced arXiv papers based on the papers in your local `.bib` files, then sends the results by email on a daily schedule using GitHub Actions. It also includes a manual workflow for a last-7-days top-10 recommendation run, and the same manual mode can be stretched with custom parameters. For example, you can search up to `1000` candidates from the last `99` days and keep the top `50` matches:

```bash
.venv/bin/python src/main.py --config config.yaml --lookback-days 99 --max-candidates 1000 --max-results 50 --dry-run --output-html output/manual_99day_top50_report.html
```

So this project works both as a daily email recommender and as a search tool that ranks arXiv papers against your own `.bib` library. For "find papers close to my research taste" style queries, it is often more useful than plain keyword-and-field filtering.

You can run it locally to generate an HTML report without sending email, or let GitHub Actions run it on schedule and send the results automatically.

This repository is designed for beginners:

- You put one or more `.bib` files under `data/`
- You choose a few arXiv categories in `config.yaml`
- GitHub Actions runs every day
- The workflow sends an HTML email with arXiv and PDF links

The current version does not use OpenAI, Claude, or any paid LLM API. It uses an open-source embedding model locally on the GitHub Actions runner.

## What This Project Does

1. Read every `.bib` file under `data/**/*.bib`
2. Keep entries that have both `title` and `abstract`
3. Fetch newly announced arXiv papers from the categories you configured, and fall back to `https://export.arxiv.org/api/query` over the last 24 hours when RSS is temporarily empty
4. Compute text embeddings for your library papers and the new arXiv candidates
5. Rank candidates by similarity to your library
6. Send an HTML email with the top matches

You can also trigger a manual weekly run that queries arXiv submissions from the last `7` days through the export API and emails the top `10` closest matches.

The current email includes:

- paper title
- score
- authors
- abstract snippet
- arXiv link
- PDF link
- the closest matching papers from your `.bib` library

The current version does not send PDF attachments. It sends links only.

## Repository Layout

```text
.
├── data/                     # Put one or more .bib files here
├── src/
│   ├── bib_loader.py
│   ├── arxiv_fetcher.py
│   ├── embedder.py
│   ├── embedding_cache.py
│   ├── recommender.py
│   ├── emailer.py
│   └── main.py
├── config.yaml               # Non-secret configuration
├── requirements.txt
└── .github/workflows/
    ├── daily.yml
    └── manual-weekly-top10.yml
```

## Before You Start

You need:

- a GitHub repository
- one email account that is allowed to send mail via SMTP
- one mailbox that will receive the daily email
- one or more `.bib` files with abstracts

Important:

- `data/*.bib` is part of the repository contents. If your bibliography is private, do not use a public repository.
- SMTP passwords, app passwords, and recipient addresses should go into GitHub Actions secrets, not into `config.yaml`, code, screenshots, or issue comments.

## Quick Start

### 1. Put your `.bib` files into `data/`

This project supports multiple `.bib` files.

Examples:

```text
data/library.bib
data/reading/ml.bib
data/reading/vision.bib
```

Minimal working BibTeX entry:

```bibtex
@article{attention2023,
  title = {A Paper Title},
  abstract = {This abstract is required for similarity matching.},
  author = {Alice Example and Bob Example},
  year = {2023}
}
```

If an entry does not contain an `abstract`, it is skipped.

### 2. Edit `config.yaml`

Start with categories that are close to your research area. Do not use too many categories at first.

Example:

```yaml
arxiv:
  categories:
    - cs.LG
    - cs.AI
    - cs.CL
  max_candidates: 80

embedding:
  model: BAAI/bge-small-en-v1.5
  batch_size: 32

ranking:
  top_k_neighbors: 5
  max_results: 15

email:
  subject_prefix: "[arXiv Daily]"
  include_pdf_links: true
  send_empty_email: false

runtime:
  data_dir: data
  output_html: output/latest_report.html
  cache_dir: .cache/recommender
```

Advice for beginners:

- Start with `2` to `4` categories
- Keep `max_candidates` around `50` to `100`
- Leave the embedding model as default first

## Email Setup: What Must Be Enabled

This project sends mail using normal SMTP login. That means your sender mailbox must allow SMTP sending.

What you usually need to enable:

- SMTP service for the sender mailbox
- an app password or authorization code if your provider uses one
- SSL or STARTTLS support

Common cases:

### Gmail personal accounts

Recommended beginner path:

1. Turn on 2-Step Verification for your Google account
2. Create an App Password
3. Use:
   - `SMTP_HOST=smtp.gmail.com`
   - `SMTP_PORT=465`
   - `SMTP_USE_SSL=true`
   - `SMTP_USER=your_gmail_address`
   - `SMTP_PASSWORD=your_16_digit_app_password`

Google official references:

- App passwords: https://support.google.com/mail/answer/185833
- Gmail in other mail clients: https://support.google.com/mail/answer/75726

Note:

- Personal Gmail is usually the easiest Google option for this repository.
- For Google Workspace accounts, organization policy may require OAuth or restrict app passwords. If you are using a work or school account, ask your admin first.

### Outlook / Microsoft 365

This repository currently uses SMTP username/password style authentication. That is not the best fit for new Microsoft 365 setups.

As of January 27, 2026, Microsoft’s official update says:

- SMTP AUTH Basic Authentication remains unchanged through December 2026
- it will be disabled by default for existing tenants at the end of December 2026
- Microsoft will announce the final removal date in the second half of 2027

Official references:

- Updated timeline: https://techcommunity.microsoft.com/blog/exchange/updated-exchange-online-smtp-auth-basic-authentication-deprecation-timeline/4489835
- Exchange Online SMTP AUTH docs: https://learn.microsoft.com/en-us/Exchange/mail-flow-best-practices/how-to-set-up-a-multifunction-device-or-application-to-send-email-using-microsoft-365-or-office-365

Practical advice:

- If you are a beginner, Gmail personal, QQ Mail, 163 Mail, or another SMTP provider with an app password is simpler.
- If you must use Microsoft 365, expect future OAuth work.

### Other providers such as QQ Mail / 163 Mail

The exact UI changes over time, but the pattern is usually:

1. Sign in to webmail
2. Enable SMTP or POP3/IMAP/SMTP service
3. Generate an authorization code
4. Use that authorization code as `SMTP_PASSWORD`

If your provider offers both the login password and a separate authorization code, use the authorization code.

## GitHub Actions Setup: What Must Be Enabled

### 1. Enable GitHub Actions for the repository

Open:

`Settings` -> `Actions` -> `General`

Recommended settings for beginners:

- `Actions permissions`: `Allow all actions and reusable workflows`
- `Workflow permissions`: `Read repository contents permission`

Why this is enough:

- this workflow only needs GitHub-authored actions such as `actions/checkout`, `actions/setup-python`, and `actions/cache`
- it does not need to write back to the repository

GitHub official docs:

- Repository Actions settings: https://docs.github.com/github/administering-a-repository/managing-repository-settings/disabling-or-limiting-github-actions-for-a-repository

### 2. If this is a fork, enable workflows in the Actions tab

GitHub official docs say:

- workflows do not run in forked repositories by default
- scheduled workflows are disabled by default in public forks
- scheduled workflows in public repositories can also be automatically disabled after 60 days of no repository activity

So after forking:

1. Open the `Actions` tab
2. Click the button to enable workflows
3. If the schedule stops later, re-enable the workflow in the Actions UI

GitHub official references:

- Events in forks: https://docs.github.com/en/actions/reference/events-that-trigger-workflows
- Disable/enable workflows: https://docs.github.com/actions/managing-workflow-runs/disabling-and-enabling-a-workflow

## Where To Put Private Information

Put private values in:

`Settings` -> `Secrets and variables` -> `Actions` -> `Repository secrets`

Create these secrets:

- `SMTP_HOST`
- `SMTP_PORT`
- `SMTP_USER`
- `SMTP_PASSWORD`
- `EMAIL_TO`

Optional secrets:

- `EMAIL_FROM`
- `SMTP_USE_SSL`

Recommended rule:

- If it is a password, token, app password, authorization code, email address, or host you do not want public, put it in `Repository secrets`
- Keep only non-secret behavior settings in `config.yaml`

Current workflow files:

- [`.github/workflows/daily.yml`](./.github/workflows/daily.yml)
- [`.github/workflows/manual-weekly-top10.yml`](./.github/workflows/manual-weekly-top10.yml)

Current config file:

- [`config.yaml`](./config.yaml)

## Example Secrets

Example for Gmail personal:

```text
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_USER=yourname@gmail.com
SMTP_PASSWORD=your_16_digit_app_password
EMAIL_TO=yourname@gmail.com
EMAIL_FROM=yourname@gmail.com
SMTP_USE_SSL=true
```

## First Manual Test

After you commit your files and secrets:

1. Open the `Actions` tab
2. Open the `arxiv-daily` workflow
3. Click `Run workflow`
4. Wait for the run to finish

What to look for in the logs:

- library loaded successfully
- arXiv papers fetched successfully
- `Saved ... library embeddings to cache ...` on the first run
- `Loaded ... library embeddings from cache ...` on later runs
- email sent successfully

If there are no recommendations and `send_empty_email: false`, the workflow can finish successfully without sending an email. That is normal.

If you want a manual weekly summary instead of the normal daily run:

1. Open the `Actions` tab
2. Open the `arxiv-weekly-manual-top10` workflow
3. Click `Run workflow`
4. Wait for the run to finish

That workflow is manual-only. It queries the last `7` days of arXiv submissions through the export API, keeps up to `500` candidates, ranks them against your `.bib` library, and emails the top `10` matches.

## Daily Schedule

The current workflow schedule is defined in:

- [`.github/workflows/daily.yml`](./.github/workflows/daily.yml)

Current cron:

```yaml
schedule:
  - cron: "30 1 * * *"
```

That means the workflow runs at `01:30 UTC` every day.

If you want a different time, edit the cron line and commit the change.

## Manual Weekly Workflow

The repository also includes:

- [`.github/workflows/manual-weekly-top10.yml`](./.github/workflows/manual-weekly-top10.yml)

This workflow has `workflow_dispatch` only. It does not run on a schedule.

Current fixed behavior:

- query arXiv papers submitted in the last `7` days
- use the export API directly instead of RSS
- score up to `500` candidates
- send the top `10` matches by email

## Model Used

This project currently uses the open-source embedding model:

- `BAAI/bge-small-en-v1.5`

It is loaded through `sentence-transformers` and runs locally on the GitHub runner.

Important:

- This is an embedding model, not a chat model
- It is used only for text similarity
- No OpenAI API key is required
- No per-token API billing is involved

Model card:

- https://huggingface.co/BAAI/bge-small-en-v1.5

## Runtime and Time Cost

The main runtime factors are:

1. dependency installation
2. PyTorch installation
3. first-time model download
4. number of `.bib` entries with abstracts
5. number of new arXiv candidates
6. GitHub Actions cache hit or miss
7. network speed to GitHub, Hugging Face, and arXiv

This repository now caches:

- Hugging Face model files
- library embeddings in `.cache/recommender`

That means:

- if your `.bib` files do not change, library embeddings are reused
- only the daily arXiv candidates need fresh embedding work

Rough runtime estimates on standard `ubuntu-latest` runners are:

- first cold run: about `5` to `12` minutes
- normal warm run with cache hit: about `1` to `4` minutes
- large libraries with thousands of papers: can be much slower

These are practical estimates, not hard guarantees.

GitHub official runner reference for standard public runners:

- `ubuntu-latest` standard runner: `4 vCPU`, `16 GB RAM`, `14 GB SSD`
- source: https://docs.github.com/en/actions/reference/github-hosted-runners-reference

## GitHub Actions Free Minutes

This part changes over time, so the numbers below are verified against GitHub Docs as of `2026-03-07`.

### Public repositories

For standard GitHub-hosted runners, GitHub Actions usage is free and unlimited in public repositories.

### Private repositories

Included monthly minutes for standard GitHub-hosted runners:

| Plan | Included minutes per month |
| --- | ---: |
| GitHub Free | 2,000 |
| GitHub Pro | 3,000 |
| GitHub Free for organizations | 2,000 |
| GitHub Team | 3,000 |
| GitHub Enterprise Cloud | 50,000 |

Notes:

- larger runners are billed separately
- if your account has no valid payment method, usage is blocked after the included quota is exhausted
- storage for artifacts and caches also has plan limits

Official references:

- Billing overview: https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions
- Included usage table: https://docs.github.com/en/billing/reference/product-usage-included

## Recommended Beginner Setup

If you want the simplest path:

1. Use a private GitHub repository if your bibliography is private
2. Put only a few `.bib` files under `data/`
3. Use `2` to `4` arXiv categories
4. Use Gmail personal with an App Password
5. Trigger the workflow manually first
6. Check that cache hits appear on the second run

## Local Run

If you want to test locally before using GitHub Actions:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python src/main.py --config config.yaml --dry-run
```

`--dry-run` writes the HTML report but skips SMTP send.

To reproduce the manual weekly top-10 workflow locally:

```bash
.venv/bin/python src/main.py --config config.yaml --lookback-days 7 --max-candidates 500 --max-results 10 --dry-run --output-html output/manual_weekly_top10_report.html
```

Useful CLI overrides:

- `--lookback-days N`: query the last `N` days through the arXiv export API instead of RSS new announcements
- `--max-candidates N`: override `arxiv.max_candidates` from `config.yaml`
- `--max-results N`: override `ranking.max_results` from `config.yaml`
- `--output-html PATH`: write the rendered report to a custom path

## Troubleshooting

### RSS returns 0 papers

There are two different cases:

- weekends or holidays, when arXiv may simply have no new announcement batch for your categories
- the RSS blank window, where the daily announcement is already visible but `rss.arxiv.org` has not caught up yet

In practice this means you can sometimes see:

- arXiv daily announcement is already online
- `RSS new papers = 0`
- a few hours later the RSS feed starts returning entries normally

This repository now handles that gap automatically:

- it still checks RSS first
- if RSS returns `0` new ids, it falls back to `https://export.arxiv.org/api/query`
- the fallback searches `submittedDate` in the last `24` hours for your configured categories

This fallback helps during the RSS propagation lag, but it does not magically create papers on days when arXiv really did not release a new batch.

### No email arrives

Check:

- whether the workflow succeeded
- whether `send_empty_email` is `false` and no recommendations were found
- whether SMTP service is enabled on the sender mailbox
- whether you used the app password or authorization code instead of the normal login password
- whether the sender mailbox blocks automated sign-in

### Workflow is visible but does not run on schedule

Check:

- whether Actions are enabled for the repository
- whether the workflow was disabled manually
- whether this is a fork and scheduled workflows were disabled by default
- whether the repository was inactive long enough for GitHub to disable scheduled workflows automatically

### The first run is very slow

That is expected if:

- dependencies are being installed for the first time
- the embedding model is downloaded for the first time
- your library cache has not been built yet

### Too many irrelevant recommendations

Usually fix this by:

- narrowing your arXiv categories
- reducing `max_candidates`
- cleaning low-quality `.bib` entries
- removing entries without meaningful abstracts

## Current Limitations

- email uses SMTP login, not OAuth
- no PDF attachments
- no HTML artifact upload in the workflow
- recommendations rely on title + abstract, not full text
- entries without `abstract` are skipped

## References

- GitHub Actions repository settings:
  https://docs.github.com/github/administering-a-repository/managing-repository-settings/disabling-or-limiting-github-actions-for-a-repository
- GitHub workflow enable/disable:
  https://docs.github.com/actions/managing-workflow-runs/disabling-and-enabling-a-workflow
- GitHub Actions billing:
  https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions
- GitHub included usage:
  https://docs.github.com/en/billing/reference/product-usage-included
- GitHub-hosted runner specs:
  https://docs.github.com/en/actions/reference/github-hosted-runners-reference
- Gmail app passwords:
  https://support.google.com/mail/answer/185833
- Gmail in other email clients:
  https://support.google.com/mail/answer/75726
- Microsoft 365 SMTP submission:
  https://learn.microsoft.com/en-us/Exchange/mail-flow-best-practices/how-to-set-up-a-multifunction-device-or-application-to-send-email-using-microsoft-365-or-office-365
- Microsoft SMTP AUTH timeline update:
  https://techcommunity.microsoft.com/blog/exchange/updated-exchange-online-smtp-auth-basic-authentication-deprecation-timeline/4489835

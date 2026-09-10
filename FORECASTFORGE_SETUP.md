# ForecastForge-bot — Local Setup & Next Manual Steps

This repo is populated from the **official Metaculus bot template**
([`Metaculus/metac-bot-template`](https://github.com/Metaculus/metac-bot-template)),
upstream commit `6dab04cdd1153a31b3f222b7a6a443b1f675b8c2` (2026-08-27).

The upstream structure is preserved. ForecastForge-specific changes so far:

- `pyproject.toml` → `name`/`description`/`authors` renamed to ForecastForge-bot
  (cosmetic; `package-mode = false`, so it does not affect dependency resolution).
- `poetry.toml` added → keeps the virtualenv inside the project (`.venv/`, git-ignored).
- This file (`FORECASTFORGE_SETUP.md`).

**No forecasting logic in `main.py` / `main_with_no_framework.py` has been changed.**

---

## 1. What this bot does

`main.py` (recommended) uses the [`forecasting-tools`](https://github.com/Metaculus/forecasting-tools)
framework. It has three run modes:

| Mode | Command | Target | Posts to Metaculus? |
|------|---------|--------|---------------------|
| `test_questions` | `poetry run python main.py --mode test_questions` | [bot-testing-area](https://www.metaculus.com/tournament/bot-testing-area/) (non-scored sandbox) | **Yes** (to the sandbox) |
| `tournament` (default) | `poetry run python main.py` | Summer FutureEval 2026 **+ MiniBench** | **Yes** (real tournament) |
| `metaculus_cup` | `poetry run python main.py --mode metaculus_cup` | Metaculus Cup | **Yes** (real) |

`publish_reports_to_metaculus=True` is hard-coded near the bottom of `main.py`
(`publish_to_metaculus = True`). Every mode publishes. To do a true dry run, set that
to `False` locally (do not commit the change if you want CI to keep publishing).

> ⚠️ As of this setup, **no forecast has been run or submitted** by ForecastForge-bot.

---

## 2. Runtime / version requirements

- **Python 3.11+** — upstream CI pins **3.11**. Local setup here was verified on the
  machine's Python 3.14; if you hit dependency issues, install 3.11 and point Poetry at it
  (`poetry env use path\to\python3.11`).
- **Poetry** (package manager used by the template). Installed here via `pip install --user poetry`
  (Poetry 2.4.3). Poetry's own docs recommend `pipx install poetry`.
- Dependencies are locked in `poetry.lock` — do not hand-edit.

---

## 3. Install / verify / run commands

```bash
# from the project root
poetry install --no-interaction --no-root      # install deps into ./.venv
poetry check                                   # validate pyproject/lock
poetry run python -m py_compile main.py main_with_no_framework.py bot_helpers.py
poetry run python main.py --help               # arg parsing sanity check
```

There is **no unit-test suite, linter, or type-checker config** in the upstream template,
so "tests" = the compile + import + `--help` checks above.

**Do NOT run `poetry run python main.py ...` (any mode) until secrets are configured and you
have decided to go live** — every mode contacts Metaculus, spends LLM credits, and posts.

---

## 4. Exact secret / environment variable names

Verified from `.env.template`, the three `.github/workflows/*.yaml` files, and
`forecasting_tools`'s default-model logic
(`forecast_bots/forecast_bot.py::_llm_config_defaults`).

### Required

| Variable | Purpose | Where to get it |
|----------|---------|-----------------|
| `METACULUS_TOKEN` | Bot account auth — read questions, post forecasts | https://www.metaculus.com/futureeval/participate/ |
| `OPENROUTER_API_KEY` | **The Metaculus-funded LLM credits key** ($100, FutureEval/MiniBench) | Provided to you by Metaculus. |

> The FutureEval free-credit programme issues an **OpenRouter** key, so it goes in
> `OPENROUTER_API_KEY`. **Confirm the variable name in the email/message that delivered your
> funded key.** If Metaculus specified a different name (e.g. a Metaculus-proxy key), use that
> exact name instead — the model-selection logic in section 5 keys off these names.

### Optional (research quality only — leave unset if you have no funded/free key for them)

`PERPLEXITY_API_KEY`, `EXA_API_KEY`, `ASKNEWS_CLIENT_ID` + `ASKNEWS_SECRET`,
`LIGHTNINGROD_API_KEY` (integrations only).

### 🚫 Must-NOT-set — personal-billing risk

**Do not set `OPENAI_API_KEY` or `ANTHROPIC_API_KEY`** anywhere (not in `.env`, not as a
GitHub Secret). See section 5 for why.

---

## 5. Funding safety — how the template picks a provider

`forecasting-tools` chooses default models **at bot startup**, based purely on **which env
vars are present**, in this priority order (from `_llm_config_defaults`):

```
OPENAI_API_KEY  >  ANTHROPIC_API_KEY  >  OPENROUTER_API_KEY  >  METACULUS_TOKEN (Metaculus proxy)
```

Implications for ForecastForge-bot:

- **Set only `OPENROUTER_API_KEY` (funded) + `METACULUS_TOKEN`.**
- **`main.py` now pins `llms={}` explicitly** (all four `openrouter/*` slugs):
  `default`/`summarizer`/`parser` = `openrouter/openai/gpt-4o` + `...gpt-4o-mini`,
  `researcher` = `openrouter/openai/gpt-4o:online`. This was done because the auto-selected
  researcher `openrouter/openai/gpt-4o-search-preview` is **not served by OpenRouter**
  (HTTP 404 "No endpoints found") and broke every research step. Perplexity Sonar was tried
  next but is **blocked by the funded key's provider allowlist** (openai / anthropic /
  google-ai-studio only), so the researcher uses the OpenAI base model plus OpenRouter's
  `:online` web-search suffix — provider stays OpenAI, live research retained. Pinning also
  makes the funded-key routing explicit and immune to future `_llm_config_defaults` changes.
- **If `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` were set, they would silently take priority**
  over the funded key and bill your personal account. That is the one and only personal-billing
  path — and it only exists if you create those secrets. Don't.
- **If the funded OpenRouter key is exhausted or invalid:** OpenRouter returns an error, the
  run fails loudly. `forecasting-tools` does **not** re-select a different provider mid-run,
  and there is **no code that buys credits or performs any payment action**. This is the
  desired "fail clearly" behaviour.
- The `METACULUS_TOKEN` Metaculus LLM proxy is **not** personal billing — it's a
  Metaculus-run allowance, used only as a last resort when *no* LLM key is set. With the
  funded OpenRouter key set, it is never used for LLM calls.
- The GitHub workflows list `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc. in their `env:`
  blocks. **Unset GitHub Secrets resolve to an empty string, which the code treats as
  "not set"** — so simply never creating those secrets is sufficient and safe. The workflow
  files are left unchanged from upstream.

Beyond the explicit `llms={}` pin above, enforcement is purely a matter of **which secrets
you create**. (Further optional hardening, not done: delete the unused `*_API_KEY` lines
from the workflow `env:` blocks.)

---

## 6. Where YOU manually place the secrets

### Locally (never committed — `.env` is in `.gitignore`)

```bash
cp .env.template .env
```

Then edit `.env` and replace the `REPLACE_ME` placeholders:

```
METACULUS_TOKEN=<your Metaculus bot token>
OPENROUTER_API_KEY=<your Metaculus-funded OpenRouter key>
```

Leave `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` **commented out / absent**.
`main.py` and `main_with_no_framework.py` call `dotenv.load_dotenv()`, so `.env` is picked
up automatically.

- ❌ Never paste a key into `main.py`, tests, README, commit messages, or logs.
- ❌ Never `git add .env`. Run `git status` before every commit and confirm `.env` is
  untracked/ignored.

### Later, in GitHub Actions (only when you decide to automate)

`Settings → Secrets and variables → Actions → New repository secret`. Names must match
exactly:

- `METACULUS_TOKEN`
- `OPENROUTER_API_KEY`
- (optional) `PERPLEXITY_API_KEY`, `EXA_API_KEY`, `ASKNEWS_CLIENT_ID`, `ASKNEWS_SECRET`

Do **not** add `OPENAI_API_KEY` / `ANTHROPIC_API_KEY`.

For the optional weekly review workflow: add repository **variable** (not secret)
`REVIEW_BOT_ENABLED=true` to enable it — it is read-only and spends nothing.

---

## 7. The safe test (run only when ready)

The official smoke test posts to the **bot-testing-area** tournament — a non-scored sandbox,
so it does not affect any real tournament standing, but it *does* spend a small amount of the
funded LLM credits and posts comments under your bot account.

```bash
poetry run python main.py --mode test_questions
```

Expected: a startup banner, forecasting logs, then `🎉 Bot submitted N forecast(s)` with
links. Verify on your bot's Metaculus profile.

To test with **zero** posting and **zero** spend, there is no built-in flag that skips the
LLM calls; the closest is setting `publish_to_metaculus = False` in `main.py` (still spends
LLM credits to generate the forecast, just doesn't upload it).

---

## 8. Enabling FutureEval / MiniBench automation (only after a successful test)

The workflow **`.github/workflows/run_bot_on_tournament.yaml`** is the one that forecasts on
**FutureEval + MiniBench**:

- Runs `poetry run python main.py` (default `tournament` mode → `CURRENT_AI_COMPETITION_ID`
  + `CURRENT_MINIBENCH_ID`).
- Schedule: `cron: "7,27,47 * * * *"` — every 20 minutes. Skips questions already forecast.
- On GitHub it is **enabled the moment Actions is enabled on the repo.** To keep it off
  until you're ready: after pushing, go to
  `Actions → Forecast on new AI tournament questions → ··· → Disable workflow`, or delete the
  `schedule:` block and keep only `workflow_dispatch`.

Recommended rollout:
1. Push repo to GitHub (private is fine), **Actions disabled** or tournament workflow disabled.
2. Add `METACULUS_TOKEN` + `OPENROUTER_API_KEY` secrets.
3. Run **`Test Bot`** workflow manually (`Actions → Test Bot → Run workflow`). Confirm forecasts land in bot-testing-area.
4. Check OpenRouter usage to confirm spend came from the funded key.
5. Only then enable **`Forecast on new AI tournament questions`**.

---

## 9. Workflow inventory

| File | Trigger | Purpose | Secrets used |
|------|---------|---------|--------------|
| `test_bot.yaml` | manual only (`workflow_dispatch`) | Smoke test → bot-testing-area | `METACULUS_TOKEN` + LLM key |
| `run_bot_on_tournament.yaml` | every 20 min + manual | **FutureEval + MiniBench** live forecasting | `METACULUS_TOKEN` + LLM key |
| `run_bot_on_metaculus_cup.yaml` | midnight every 2 days + manual | Metaculus Cup | `METACULUS_TOKEN` + LLM key |
| `review_bot.yaml` | weekly Mon 06:00 (needs `REVIEW_BOT_ENABLED=true`) + manual | Read-only scoring of resolved forecasts — no spend, no posting | `METACULUS_TOKEN` |

Nothing in the workflows needs changing for ForecastForge-bot **except** the choice of which
secrets you create (see sections 4–5). Keep them as-is otherwise.

---

## 10. Credentials safety checklist

- [ ] `.env` exists locally, contains real keys, and is **git-ignored** (`git check-ignore .env` prints `.env`).
- [ ] `git log -p` / tracked files contain **no** real key (only `REPLACE_ME` placeholders in `.env.template`).
- [ ] `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` set **nowhere**.
- [ ] Funded key added only by you, only in `.env` locally and GitHub Secrets later.
- [ ] No secret printed to logs or terminal.
- [ ] Repo not pushed until you have reviewed it.

# Credential-Attack (Phishing) Domain Detector — Guideline & Tutorial

**Version:** 1.0 (2026-11-02) · **Language:** Python 3.10+ · **Cloud required:** none

A local, two-layer detector that flags domains impersonating banks, email
providers and other brands. Layer 1 is a deterministic whitelist of sanctioned
brand domains; Layer 2 is logistic regression over 15 domain heuristics.
Everything runs on `127.0.0.1` — no account, no API key, no telemetry.

---

## 1. What you need

| Item | Requirement |
|---|---|
| OS | Linux, macOS, or Windows (Python available) |
| Python | 3.10 or newer (`python3 --version`) |
| Internet | **only** on the first run (downloads ~20 MB of public datasets; cached forever after) |
| Hardware | any laptop — training takes ~4 seconds |

## 2. Setup (once, ~1 minute)

```bash
# from the project folder
python3 -m venv .venv
# activate (pick your shell)
source .venv/bin/activate        # bash/zsh
.venv\Scripts\activate           # Windows PowerShell/cmd
pip install scikit-learn         # training only — the runtime needs nothing else
```

## 3. Quickstart (the whole project in 3 commands)

```bash
python train.py            # 1. fetch data (first time) → train → save model + metrics
python test_detector.py    # 2. self-check: prints ALL CHECKS PASSED
python app.py              # 3. web UI + API on http://127.0.0.1:8321
```

Open **http://127.0.0.1:8321** in your browser. Try these two and watch the
difference:

- `paypal-verify-secure.cf` → **PHISHING** (suspicious TLD + brand impersonation + credential words)
- `accounts.wellsfargo.com` → **BENIGN** (whitelist layer: a real bank's own subdomain is sanctioned)

Press `Ctrl+C` in the terminal to stop the server.

## 4. How a decision is made

Every `/api/check` answer is built in two layers:

1. **Whitelist check** — is the domain one of a brand's *own* domains or a
   subdomain of one (`BRAND_REAL` in `features.py`)? If yes → BENIGN, done.
   Rationale: a phisher cannot use `accounts.google.com` without a Google
   compromise.
2. **Model check** — 15 numeric features (length, dot/hyphen/digit counts,
   character entropy, suspicious-TLD flag, brand-impersonation flag,
   credential-word flag, and two TLD-interaction features) go through a
   logistic-regression weight vector stored in `models/model.json`.
   Output = probability; ≥ 0.5 → PHISHING.

The UI shows the **top 5 feature signals** (feature value + its weighted
contribution, positive = pushes toward PHISHING). That breakdown is the
model explaining itself — read it, trust it, or debug it.

## 5. Using the JSON API

```bash
curl -X POST http://127.0.0.1:8321/api/check \
  -H 'Content-Type: application/json' \
  -d '{"url":"chase-secure-update.xyz"}'
```

Example response:

```json
{
  "input": "chase-secure-update.xyz",
  "label": "PHISHING",
  "prob": 1.0,
  "features": [
    {"name": "tld_suspicious", "label": "suspicious TLD", "value": 1, "signal": 7.3},
    {"name": "brand_x_tld",    "label": "brand impersonation on suspicious TLD", "value": 1, "signal": 2.1},
    ...
  ]
}
```

| Endpoint | Method | Purpose |
|---|---|---|
| `/api/check` | POST `{"url": ...}` | classify a domain/URL |
| `/api/health` | GET | liveness check → `{"ok": true}` |
| `/` | GET | the web UI |

Bad input (not domain-shaped) → HTTP 400 `{"error": "bad input"}`.
URLs with paths or ports are normalized to the host before scoring.
**Security note:** the server binds to loopback only; it never sends data
anywhere. Don't expose the port to a network without a real auth layer.

## 6. Retraining & updating

- **Re-run any time:** `python train.py` — data files in `data/` are only
  downloaded when missing, so a retrain is offline after the first run.
- **Refresh the corpora:** delete the raw lists in `data/` (keep
  `legit.json` if you like it) and re-run; `train.py` refetches from the
  sources listed in `data/SOURCES.txt`.
- **Re-run the check after every retrain:** `python test_detector.py`.
- Metrics land in `data/eval_report.json`; the current held-out numbers are
  precision 0.995 / recall 0.882 / AUC 0.967 (19,064 phishing vs 1,001 legit).

## 7. Extending it (the intended upgrade paths)

1. **Add a brand** — 2 lines in `features.py`:
   `BRAND_REAL["mybank"] = ("mybank.com", "mybank.co.uk")` → the whitelist
   and the impersonation logic both pick it up. Retrain is optional (the
   model weights don't change).
2. **Lower the threshold** — for a "warn aggressively" mode set the 0.5 cut
   in `app.py:score()` to e.g. 0.3 (raises recall, adds false alarms).
3. **Page-level features** — fetch the landing HTML and score form-action
   host / keyword injection. This is the biggest lever on the non-brand tail.
4. **Typosquatting** — Levenshtein distance + homoglyph check against a brand
   domain list, for lookalikes of brands not in the whitelist.

## 8. Troubleshooting

| Symptom | Fix |
|---|---|
| `Port 8321 already in use` | `python app.py 8322` (any free port) or `pkill -f "app.py 8321"` |
| `ModuleNotFoundError: sklearn` | you're not using the venv — `source .venv/bin/activate` |
| First run hangs on fetch | the public corpora changed/blocked — delete the partial file in `data/` and check the URL in `train.py` |
| A brand I trust gets flagged | add its real domains to `BRAND_REAL` (§7.1), or check the feature breakdown — you'll see exactly which signal won |
| UI shows `bad input` | only host/domain strings work in the box, e.g. `paypal.com` or `https://paypal-verify.cf/x` |

## 9. Project layout

```
credential-attack_detector/
├── features.py          # feature extraction + whitelist (pure stdlib)
├── train.py             # fetch → clean → fit → models/model.json + metrics
├── app.py               # stdlib HTTP server: UI + /api/check
├── test_detector.py     # one runnable self-check
├── static/index.html    # the web UI (single file, no build step)
├── models/model.json    # trained weights — dependency-free
├── data/                # corpora + SOURCES.txt + eval_report.json
├── data/SOURCES.txt     # provenance: every dataset, URL, row count, date
└── README.md            # short version of this
```

## 10. Honest limitations

- **Domain-level only.** Phishing pages hosted *inside* legitimate platforms
  (`x.weebly.com`) are out of scope (they're filtered from training, not "missed by luck").
- **Recall ≈ 0.88** on the non-brand, random-string tail — the weak slice.
- **Whitelist is a fixed list**; typosquats of unlisted brands rely on generic features only.
- **Not a production WAF.** It's an educational detector: fast, explainable,
  local — use it to *learn and demo*, with the upgrade paths above as the roadmap.

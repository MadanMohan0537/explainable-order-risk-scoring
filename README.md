# Explainable Order Risk Scoring (FraudShield)

**A rules-based, explainable e-commerce transaction risk-scoring API and inspection UI on Cloudflare Workers, with a separate offline classification experiment.**

[Live demo](https://e-commerce-fraud-detection.madanmohanlearning.workers.dev/) · Previously named `E-Commerce-Fraud-Detection`

> **What it is and is not:** the online endpoint uses **deterministic rules**. Scores are heuristic values, **not calibrated fraud probabilities**. The Python training script is an offline experiment; its model is **not** served by the Worker.

---

## The problem

Merchants and risk analysts need to understand *why* a transaction looks risky, not just get a number. Opaque scores are hard to tune, hard to explain to customers who are declined, and hard to audit.

## Who it is for

Analysts, product managers, and engineers exploring how transparent risk rules behave before investing in a trained model.

## Approach

- Accept ten numeric inputs (amount, account age, transactions today, distance from home, hour of day, weekend flag, international flag, payment method, device type, and failed attempts) plus short location and date strings. Numbers outside the ranges in the Worker are rejected.
- Apply transparent rules and return the **contributing risk factors with severities**, plus a risk label. Amount, account age, same-day count, distance, a home-state versus transaction-state mismatch, hour, the international flag, and failed attempts can add risk. The score ignores weekend, payment method, device type, city, and date.
- Keep the rule engine and any trained model **separate**, so each can be evaluated on its own.

## What exists today

- Worker API (`POST /predict`, `GET /health`) and a browser form that scores one transaction at a time. The form can express a high amount, a new account, an unusual hour, a long distance, or failed attempts; each submission replaces the previous result.
- An offline supervised-classification experiment in `training/`.
- No automated tests and no CI yet. `npm run check` runs `node --check` on `worker/index.js` only.

## What you can try

- Submit transaction attributes and inspect contributing risk factors.
- Edit the inputs and submit again to compare cases such as a high amount, a new account, an unusual hour, a long distance, or failed attempts.
- Run the Worker locally with a Wrangler release that accepts this project's compatibility date, or deploy it to your own Cloudflare account with `npx wrangler deploy`.
- Explore supervised classification separately with the training script.

## Quick start

Use Node.js 22 or newer and npm. Wrangler 4.120.0 and 4.151.3 both require that Node version.

```bash
git clone https://github.com/MadanMohan0537/explainable-order-risk-scoring.git
cd explainable-order-risk-scoring
npm install
npm run check
npx --yes wrangler@4.151.3 dev
```

`npm run check` runs `node --check` on `worker/index.js`. `package.json` lists no runtime dependencies. Its `dev` script pins Wrangler 4.120.0 and calls it through `npx` (the first run asks you to confirm that install). Publishing is covered under Deployment and does not use that pin.

`npm run dev` does not stay up. `compatibility_date` in `wrangler.jsonc` is `2026-08-13`, and Wrangler 4.120.0's local runtime only accepts dates through `2026-08-08`. `npx --yes wrangler@4.151.3 dev` reads the same config and serves http://localhost:8787. This README leaves `wrangler.jsonc` and the Worker name `fraudshield` as they are.

## API

### POST /predict

Send a JSON object with transaction features. Omitted numeric fields use the defaults in [worker/index.js](worker/index.js). Omitted location and date strings are empty. Values outside the allowed ranges return a validation error.

```bash
curl http://localhost:8787/predict \
  -H 'Content-Type: application/json' \
  -d '{"transaction_amount":6000,"account_age_days":5,"num_transactions_today":8,"distance_from_home_miles":2445,"hour_of_day":2,"is_international":1,"failed_attempts":3}'
```

| Field | Meaning |
| --- | --- |
| `fraud` | Whether the heuristic score exceeds 0.5 |
| `probability` | Rounded heuristic score, capped at 0.99; not a calibrated probability |
| `confidence` | Risk label derived from score thresholds |
| `risk_factors` | Objects containing `factor` and `severity` |
| `mode` | `rules` |
| `disclaimer` | Prototype decision-aid boundary |

The example activates several rules; inspect the actual response rather than relying on an illustrative score. The service keeps no cross-request history; velocity fields are supplied by the caller.

`GET /health` returns a small JSON status document (`status`, `runtime`, `mode`).

## Offline model experiment

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
python -m pip install -r requirements-training.txt
python training/train_model.py
```

Run that from the repository root. The script reads `Fraudulent_E-Commerce_Transaction_Data.csv` from the current working directory. If the file is missing, it runs `kaggle datasets download -d shriyashjagtap/fraudulent-e-commerce-transactions --unzip`, which needs Kaggle CLI credentials in addition to the `kaggle` package listed in [requirements-training.txt](requirements-training.txt). It writes `model/fraud_model.pkl` under the current working directory. The Worker leaves that file unused. This repository has no `app.py`, even though the script prints that command when it finishes. The training script's distance column is kilometers; the Worker scores miles.

## Repository map

| Location | Purpose |
| --- | --- |
| [worker/index.js](worker/index.js) | Feature validation, risk rules, HTTP routes |
| [public](public/) | Browser application served by the Worker |
| [templates/analyze.html](templates/analyze.html) | Data-analyzer page left in the tree. The Worker publishes only `public/` |
| [training/train_model.py](training/train_model.py) | Offline classification experiment |
| [requirements-training.txt](requirements-training.txt) | Python training dependencies |
| [package.json](package.json) | `check` and local `dev` scripts |
| [wrangler.jsonc](wrangler.jsonc) | Worker deployment configuration (`name` is `fraudshield`) |

## Deployment

Deploy on Cloudflare Workers from the repository root. Authenticate with the Wrangler CLI, then run:

```bash
npx wrangler deploy
```

That command reads [wrangler.jsonc](wrangler.jsonc) as it is:

| Setting | Value |
| --- | --- |
| Worker name | `fraudshield` (leave this unchanged) |
| Entry point | `worker/index.js` |
| `compatibility_date` | `2026-08-13` |
| Assets | directory `./public`, binding `ASSETS`; `/predict` and `/health` run in the Worker before assets |
| Other bindings | none |
| Secrets, vars, routes, or custom domains | none |

`compatibility_date` is a Workers runtime date in that file, not a Wrangler package version. Keep the Worker name `fraudshield`. The config does not attach https://e-commerce-fraud-detection.madanmohanlearning.workers.dev/; that address is the existing demo, and this deploy does not rename it. Observability is enabled.

## How I would measure it next (not yet done)

- Precision/recall and false-positive rate of the rules on a labeled, permitted dataset.
- Calibration of the score before calling it a probability.
- Latency of `/predict` under load.

## Limitations

No published real-world validation, calibrated probability, or latency benchmark. It should not approve or decline payments on its own. Production use would need representative labeled data, threshold calibration, monitoring, authentication, abuse controls, and human review.

## Contributing and license

Useful contributions: fixture-backed rule tests or a reproducible model evaluation. No license file is included yet; choose one before redistributing the code.

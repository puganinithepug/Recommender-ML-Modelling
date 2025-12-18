# Recommender ML System Project

Produced on GitLab McGill MIMI server. This is a collaborative project, credit to members.

Members: Élodie Côté-Gauthier, Daria Vorsina, Ziling Cheng, Sarah Al Taleb, Evan Jiang


## Movie Recommendation System Project Overview

This repository contains a collaborative filtering–based movie recommender system. The project demonstrates training, evaluation, and deployment of recommendation models using the Surprise library.

We containerize the inference service with Docker.

## Project Structure

- **data/** – Contains extracted datasets (`explicit_ratings_from_kafka.csv`, `users.csv`, `movies.csv`).
- **models/** – Stores trained recommendation models (`svd_model.pkl`).
- **monitoring/** – Includes Grafana, Prometheus, alert configurations for online evaluation and system metrics.
- **reports/** – Contains milestone reports.
- **scripts/** – Scripts for model training, offline and online evaluation.
- **tests/** – Data quality and integration tests.
- **app.py** – Flask inference API for movie recommendations.
- **Dockerfile** – Builds containerized deployment environment.
- **requirements.txt** – Lists Python dependencies.


## To get started
- Clone the repository:
``` bash
    git clone
    cd Recommender-ML
```
- Set up:
```bash
# Build the Docker image
docker build -t comp585-recommender:m2 .
```

- Set pre-push hook locally to run tests automatically before pushing:
```bash
    ./scripts/setup-hooks.sh
```

## Start recommender service inside Docker container with:
```bash 
docker run -p 8080:8080 comp585-recommender:m2
```
- Open in browser or run commands:

 To do a health check on the container, and obtain a recommendation:

## Get Recommendations
```bash
curl http://fall2025-comp585-2.cs.mcgill.ca:8080/recommend/{USER-ID}
```
Or
## Health check
curl http://fall2025-comp585-2.cs.mcgill.ca:8080/health

# Run all tests inside the container
```bash
docker run --rm comp585-recommender:m2 pytest -q tests/
```

## Automated retraining & deployment

Use the orchestration script to pull new Kafka ratings, retrain SVD (inside the pipeline container), run offline evaluation, and roll out via blue/green:

```bash
# Dry-run (prints the steps without executing)
python scripts/automate_release.py

# Execute the full pipeline
python scripts/automate_release.py --execute
```

During execution the script:

1. Runs `scripts/extract_explicit_ratings.py` (stream mode) to append fresh ratings.
2. Calls `docker run --rm comp585-recommender:pipeline ...` for each ML step (`split_train_test_set.py`, `train_surprise_models.py`, `offline_evaluation.py`), so all model code runs under Python 3.10 with the correct SciPy/Surprise stack while the host stays clean.
3. Builds and deploys the new version with `docker compose` (recommender-blue/green) on the host.
4. Performs staged traffic shifts (25% → 50% → 75% → final) using static nginx configs (`nginx-blue-primary.conf`, `nginx-equal.conf`, `nginx-green-primary.conf`) and `/health` + online KPI checks, rolling back if thresholds fail.

Key flags:

- `--skip-ingest` – reuse the existing CSV (skip Kafka).
- `--test-size`, `--rmse-max`, `--precision-min`, etc. – control Surprise split and offline gates.
- `--shift-wait` (default 3600s) – soak time between staged traffic shifts.
- `--online-metric-min` (default 0.35) – minimum positive rating rate before advancing stages.
- `--stop-old-on-success` – stop the old color’s container after the rollout.

When to use specific flags:

- **One-off bootstrap**: run `python scripts/extract_explicit_ratings.py --mode historical --output-file data/explicit_ratings_from_kafka.csv` once to populate the CSV with the backlog. Thereafter, let the pipeline append new ratings via stream mode (default).
- **Quick canary cycles**: add `--shift-wait 60` to shorten each phase to 60 seconds (useful for demos). For production cadence, stick with the default 3600s.
- **Diagnostics**: pass `--skip-ingest` if you only want to retrain/deploy using the current data. Pass `--rmse-max`/`--precision-min` overrides to temporarily widen/tighten the offline gate.
- **Custom KPI gating**: override `--online-metrics-url`, `--online-metric-name`, and `--online-metric-min` if you deploy the monitoring stack elsewhere or want a different metric (e.g., `online_user_engagement_rate`).
- **Disable staged shift**: if you ever need an instant blue→green swap, pass `--staged-shift False` (or `--no-staged-shift`); the script will deploy the idle color and immediately replace the old container.

Schedule `python scripts/automate_release.py --execute` via cron/CI every 3 days to meet the milestone requirement. The script writes release records to `reports/release_history.jsonl` for provenance.


                               ┌───────────────────────────┐
                               │   Kafka (movielog2)       │
                               └────────────┬──────────────┘
                                            │
                            ingest ratings  │
                                  ┌─────────▼──────────┐
                                  │ Host: automate_    │
                                  │ release.py runner  │
                                  │ (Python script)    │
                                  └─────────┬──────────┘
                                            │
                             docker run     │ 1) split/train/eval
                                            │
                  ┌─────────────────────────▼────────────────────────┐
                  │  comp585-recommender:pipeline container          │
                  │  (Python 3.10 + Surprise/SciPy)                  │
                  │  • scripts/split_train_test_set.py               │
                  │  • scripts/train_surprise_models.py              │
                  │  • scripts/offline_evaluation.py                 │
                  └─────────────────────────┬────────────────────────┘
                                            │
                         OK? offline metrics│
                                  ┌─────────▼──────────┐
                                  │ Host Docker daemon │
                                  │ • docker compose   │
                                  │   build/control    │
                                  └─────────┬──────────┘
                                            │
                        build+deploy        │                staged traffic
                                            │                (nginx configs)
      ┌────────────────────┐      ┌─────────▼───────────┐        ┌─────────────────────┐
      │ recommender-blue   │◀────▶│ recommender-        │◀──────▶│ recommender-load    │
      │ container (old     │      │ green container     │        │ balancer (nginx)    │
      │ model)             │      │ (new model)         │        │ static configs:     │
      └────────────────────┘      └─────────▲───────────┘        │  blue-primary       │
                                            │                    │  equal              │
                                 stop old   │                    │  green-primary      │
                                            │                    └────────┬────────────┘
                                            │                             │
                                            │ Health & KPI checks         │
                                            │ (from monitoring stack)     │
                                            │                             │
              ┌─────────────────────────────┴─────────────────────────────▼─────────────┐
              │        Monitoring docker-compose stack                                  │
              │  • Prometheus (scrapes nginx /metrics, online exporter, blackbox)       │
              │  • Grafana (dashboards on Prometheus data)                              │
              │  • Blackbox exporter (probes nginx /health)                             │
              │  • online_evaluator container (runs scripts/online_evaluation.py,       │
              │    reads Kafka & container logs every few minutes, writes               │
              │    /metrics-data/online_metrics.json via shared volume)                 │
              │  • online_metrics_exporter container (serves online_metrics.json on     │
              │    http://localhost:9108/metrics for Prometheus + pipeline KPI check)   │
              └─────────────────────────────────────────────────────────────────────────┘

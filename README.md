# AI risk sentiment event study (September 8-30, 2026)

This repository studies the short reaction to the September 8, 2026 AI-risk discussion involving Anthropic employees. It uses a transparent, deliberately narrow design: X posts establish the event; dated Google News RSS headlines provide the reproducible daily coverage proxy; and Yahoo Finance adjusted closes supply returns for an equal-weight AI basket.

## Separate public-social collectors

- `Public_Bluesky_Collection.ipynb` collects a query-defined Bluesky public-post sample.
- `Public_HackerNews_Collection.ipynb` collects a query-defined Hacker News stories/comments sample.
- `X_Full_Archive_Sentiment_Analysis.ipynb` is an optional X full-archive implementation. It requires endpoint entitlement and usage credits; its bearer token belongs in the ignored `secrets/x_bearer_token.txt` file or `X_BEARER_TOKEN` environment variable.

Both cache raw responses under ignored `data/raw/` and are deliberately separate from the original news-and-equities event-study notebook. Reddit is not collected because its Data API requires OAuth; YouTube comments are not collected because its Data API requires a key.

## Deliverables

- `AI_Sentiment_Event_Study.ipynb` - data collection, cleaning, charting, return construction, and Granger tests.
- `Public_Bluesky_Collection.ipynb`, `Public_HackerNews_Collection.ipynb`, and `X_Full_Archive_Sentiment_Analysis.ipynb` - separate optional social-source implementations.
- `REPORT.md` - 5-6 page-equivalent research report with results, caveats, and sources.
- `AI_USE.md` - AI-use disclosure.

## Reproduce

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/python -m ipykernel install --user --name ai-event-study
.venv/bin/python -m nbconvert --to notebook --execute --inplace --ExecutePreprocessor.kernel_name=ai-event-study AI_Sentiment_Event_Study.ipynb
```

The notebooks cache their public inputs in `data/raw/` and use September 8–30, 2026 (`END = "2026-10-01"`, which is exclusive). Re-run every table and figure together whenever the date range or source cache changes.

## Important interpretation constraint

The study has only 16 U.S. trading sessions and uses headline sentiment, not a firehose or a representative random sample of X posts. Granger results are predictive-lag diagnostics, **not causal evidence**, and should be interpreted as low-power exploratory results.

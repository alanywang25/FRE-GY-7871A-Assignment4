# AI risk sentiment event study (September 8-30, 2026)

This repository studies the short reaction to the September 8, 2026 AI-risk discussion involving Anthropic employees. It uses a transparent, deliberately narrow design: X posts establish the event; dated Google News RSS headlines provide the reproducible daily coverage proxy; and Yahoo Finance adjusted closes supply returns for an equal-weight AI basket.

## X-post collector

- `TwitterAPIio_Archive_Sentiment_Analysis.ipynb` collects a dated, query-defined X-post sample through TwitterAPI.io. Its API key belongs in ignored `secrets/twitterapi_io_key.txt` or `TWITTERAPI_IO_KEY`.

Both notebooks cache raw inputs under ignored `data/raw/` and are deliberately separated by data source.

## Deliverables

- `AI_Sentiment_Event_Study.ipynb` - data collection, cleaning, charting, return construction, and Granger tests.
- `TwitterAPIio_Archive_Sentiment_Analysis.ipynb` - X-post collection, cleaning, charting, return construction, and Granger tests.
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

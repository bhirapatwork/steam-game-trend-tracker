# Steam Game Trend Tracker
Author: bhirapat (bhirapat.work@gmail.com) — portfolio project for Data Engineer/Analyst internship

## About me
- I'm a student learning data engineering. Explain what you do and why in Thai, in simple terms.
- Before writing large code, explain the plan first.
- Code comments, docstrings and commit messages in English.

## Tech rules
- Python 3.12, manage packages with uv only (uv add / uv run). Never use pip directly.
- Source code in src/steam_tracker/, SQL in sql/, tests in tests/, Streamlit app in app/
- HTTP: httpx with timeout + tenacity retry with backoff; sleep between calls to respect rate limits
- Storage: raw data as Parquet in data/raw/, transformations with DuckDB SQL
- Store timestamps in UTC; display in Asia/Bangkok time
- Run ruff and pytest before every commit
- Conventional commits (feat:, fix:, docs:, chore:, test:)
- Never commit .env or secrets

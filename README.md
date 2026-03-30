# Hermes Research Corpus

Multi-topic research repository built by Hermes agent.

## Topics

- `topics/methylene-blue/` — Methylene blue health benefits + IR interaction

## Structure (per topic)

Each topic folder contains:
- `daily/` — Daily digests (YYYY-MM-DD.md)
- `sources/` — Raw source links and notes
- `summaries/` — Formatted summaries by subtopic
- `papers/` — arXiv paper references

## Usage

Daily crons run research searches and save findings. Pull to sync.

To add a new topic: create folder under `topics/` and set up a new cron job.

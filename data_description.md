# Data Description — WhatsApp Fact-Check Bot v2

## Overview

This project does **not train on a large labeled dataset** — it relies on pre-trained NLI and embedding models. The "data" layer consists of:

1. **A curated fact-check corpus** (CSV / Parquet) used for semantic retrieval of similar past claims.
2. **Live evidence** fetched at inference time from Wikipedia and DuckDuckGo.
3. **Sample test claims** for evaluation and demo purposes.

---

## Fact-Check Corpus

### Location
```
data/corpus/factcheck_corpus.csv
data/corpus/factcheck_corpus.parquet   (generated on first run)
```

### Schema

| Column | Type | Description |
|--------|------|-------------|
| `fact_id` | string | Unique identifier (e.g. `fc_00001`) |
| `claim_text_original` | string | Original claim as received |
| `claim_text_normalized` | string | Preprocessed/translated English version |
| `verdict_label` | string | `supported`, `refuted`, `mixed`, `misleading`, `not_enough_evidence` |
| `source_name` | string | Publisher name (e.g. WHO, Reuters, ISRO) |
| `source_url` | string | URL of the fact-check article |
| `article_title` | string | Title of the source article |
| `published_date` | string | ISO date (YYYY-MM-DD) |
| `article_text` | string | Full or partial article body (optional) |
| `evidence_snippets` | string | Key sentences supporting the verdict |
| `language` | string | `en` (all entries normalized to English) |
| `tags` | string | Comma-separated tags: `demo`, `auto-learned`, `health`, etc. |

### Train / Val / Test Split

This corpus is used only for **retrieval display** (showing similar past claims to the user) — NOT for making verdict decisions. There is therefore no train/val split for the corpus itself.

For **evaluation of the verdict engine**, we use a manually curated test set of 10 labeled claims (see `sample_inputs/test_claims.json`).

| Split | Size | Notes |
|-------|------|-------|
| Corpus (retrieval) | 3 demo + growing | Auto-seeded on first run |
| Evaluation test set | 10 claims | Manually labeled ground truth |

### Preprocessing Applied to Corpus

1. Script detection (Devanagari / Bengali / Hinglish / Latin)
2. WhatsApp noise removal (forwarded banners, URLs, punctuation)
3. Indian alias substitution (e.g. "NaMo" → "Narendra Modi")
4. Translation to English via `facebook/nllb-200-distilled-1.3B`
5. Claim normalization stored in `claim_text_normalized`

### Demo Seed Data (built-in, no download required)

Three seed entries are auto-created if the corpus is empty:

| Claim | Verdict | Source |
|-------|---------|--------|
| Drinking cow urine cures COVID-19 | refuted | WHO |
| India launched Chandrayaan-3 to the Moon in 2023 | supported | ISRO |
| 5G towers spread coronavirus | refuted | Reuters |

---

## Live Evidence Sources (Fetched at Runtime)

| Source | Method | Notes |
|--------|--------|-------|
| Wikipedia | `wikipedia` Python library | 4-sentence summary, English |
| DuckDuckGo | `ddgs` library, text search | Snippets only (no full-page fetch by default) |
| Web articles | `requests` + `BeautifulSoup` | Fallback only if snippets insufficient |

**Trusted domains** (prioritized in display):
```
who.int, mohfw.gov.in, pib.gov.in, cdc.gov, nih.gov,
reuters.com, apnews.com, bbc.com, indianexpress.com, thehindu.com,
factcheck.org, snopes.com, altnews.in, boomlive.in
```

---

## Dataset Links

See `dataset_links.txt` for links to external datasets.

The key pre-trained models used (downloaded automatically from HuggingFace):

| Model | HuggingFace URL |
|-------|----------------|
| NLLB-200 (translation) | https://huggingface.co/facebook/nllb-200-distilled-1.3B |
| RoBERTa-large-MNLI (NLI) | https://huggingface.co/roberta-large-mnli |
| all-MiniLM-L6-v2 (embeddings) | https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2 |
| GoEmotions RoBERTa | https://huggingface.co/SamLowe/roberta-base-go_emotions |

---

## Sample Inputs

See `sample_inputs/` for representative examples used in demo and evaluation.

```
sample_inputs/
├── test_claims.json        ← 10 labeled claims for evaluation
├── hindi_sample.txt        ← Example Hindi Devanagari input
├── hinglish_sample.txt     ← Example Romanized Hindi input
└── whatsapp_forwarded.txt  ← Example WhatsApp forwarded message with noise
```

# Decision Packet: Migrate or Stay on `google-bert/bert-base-uncased`

**Purpose:** Q3 OKR — migrate-or-stay call for the search-relevance service's masked-language-model backbone.
**Status:** Assembled for leadership review. All numbers below are live Hub/API pulls, not estimates.

---

## 1. Backbone Health (what we run today)

`google-bert/bert-base-uncased` — [https://hf.co/google-bert/bert-base-uncased](https://hf.co/google-bert/bert-base-uncased)

| Metric | Value |
| --- | --- |
| Downloads (30-day window) | **3,270.1M** (~3.27B) |
| Likes | **3,051** |
| Task | fill-mask (`AutoModelForMaskedLM`) |
| Parameters | 110.1M |
| License | Apache-2.0 |
| Last updated on Hub | 19 Feb 2024 |

**Read:** Adoption is enormous and the model is far from abandoned by the ecosystem — it remains the default teaching/scaffolding backbone. The caveat is that the weights themselves have been static since Feb 2024: the backbone is "healthy" in usage terms but frozen in capability terms.

---

## 2. Modern Alternatives (like-for-like, fill-mask / classification workloads)

All three shortlisted models are fill-mask backbones usable for the same masked-LM + classification pipeline we run.

| # | Model | Downloads | Likes | Params | License | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| — | `google-bert/bert-base-uncased` (current) | 3,270.1M | 3,051 | 110.1M | Apache-2.0 | Baseline |
| 1 | [`answerdotai/ModernBERT-base`](https://hf.co/answerdotai/ModernBERT-base) | **69.6M** | **1,145** | 149.7M | Apache-2.0 | 2024 architecture; long-context (8k); strong on retrieval/classification |
| 2 | [`answerdotai/ModernBERT-large`](https://hf.co/answerdotai/ModernBERT-large) | **12.9M** | **500** | 395.9M | Apache-2.0 | Highest quality of the family; 3.6× the parameters of our current backbone |
| 3 | [`microsoft/deberta-v3-base`](https://hf.co/microsoft/deberta-v3-base) | **77.3M** | **468** | — | MIT | Proven classification performer; Hub card untouched since Sep 2022 |

Reference only (not shortlisted): `FacebookAI/roberta-base` — 660.7M downloads / 699 likes. Kept out of the shortlist because it is an era-matched (2019) peer of BERT, not a modern replacement.

**Read:** ModernBERT-base is the standout candidate — 1,145 likes against 69.6M downloads is a strong like-per-download ratio, and it is the modern architectural replacement explicitly positioned for fill-mask/retrieval workloads. DeBERTa-v3-base remains the safe "same-size, better quality" option. ModernBERT-large is the quality ceiling but at a 3.6× parameter cost that needs a serving-cost argument before adoption.

---

## 3. Community Signal (huggingface/transformers)

### Issues

**About ModernBERT (the strongest alternative):**

- **#35385 — "Support modernBERT for encoder-decoder models"** (open, since Dec 2024): `EncoderDecoderModel.from_encoder_decoder_pretrained` fails with ModernBERT because `AutoModelForCausalLM` does not recognize `ModernBertConfig`. The reporter's motivation is explicit: "ModernBert has a better performance and a longer context length." Practical impact for us: ModernBERT is currently *encoder-only* in the standard wiring — fine for our fill-mask/classification path, a blocker if we ever need seq2seq.
- **#49106 — "ModernBERT: local-attention window is computed from sequence index, not `position_ids`, so non-monotonic positions get a silently different window"** (open): the alternating local-attention layers build their sliding window from sequence index while the rotary embeddings use `position_ids`, so any non-monotonic position layout (e.g., shared-prefix packing) silently gets a different window. A custom 4D mask under `sdpa`/`eager` gives local layers *no* window at all, with no warning raised. This matters for us specifically if we batch multiple queries per document into one pass.

**About the current backbone:**

- **#5421 — "What to do about this warning message: 'Some weights of the model checkpoint at bert-base-uncased were not used when initializing BertForSequenceClassification'"** (closed, 57 comments, 112 👍 — the most-relevant bert-base-uncased issue by reactions): the canonical "unused MLM head weights" warning when loading `bert-base-uncased` into `BertForSequenceClassification`. Answered as expected behaviour, but with 112 reactions it is the single most-felt pain point of using this checkpoint as a classification backbone — our own team has hit this exact warning.
- **#12941 — "OSError: Can't load config for 'bert-base-uncased'"** (closed): historical 403/forbidden errors fetching `config.json` from the Hub. Operational risk pattern (checkpoint retrieval), not an architecture problem.
- **#12241 — "Modify BERT encoder layers?"** (open since June 2021, still open): a long-standing unanswered request for guidance on customizing BERT's encoder layers — indicative of the "frozen, legacy" nature of the backbone rather than a defect.

### Discussions

The `huggingface/transformers` **Discussions** area returned **zero threads and zero discussion categories** for this account — no threads about deprecating classic BERT or moving to modern replacements were visible. I am stating that plainly rather than padding: there is no Discussion-level signal either way. The closest signal is issue #43364 (closed), where a whole-word-masking crash report ends with the note that `DataCollatorForWholeWordMask` "is going to be deprecated" — a maintenance-direction hint, not a deprecation of the BERT architecture itself.

---

## 4. Our Telemetry (Weights & Biases Weave)

- The account's only entity with projects is **`hezil6643-modelbest`** (26 projects; the personal entity `hezil6643` has none).
- **All 26 projects were checked for Weave traces. Every single one returned `total_count: 0`.**
- The most relevant candidates were additionally queried in detail (trace + cost): `model-review`, `model-audits`, `model-sourcing`, `sequence-models-onboarding`, `simulated_env_tau2`, `platform-audit`, `genai-ops-review`, `platform-review` — each returned `total_traces: 0`, no time range, and no cost data.
- Corroborating evidence: the "Paper Citations" tab of our `LLM Pre-training Data` spreadsheet carries `W&B Project` and `Weave Traces` columns that are entirely blank — consistent with no telemetry having been captured.

**Plain statement:** as of this packet there are **no Weave traces — and therefore no trace counts or costs — in any project our account can see.** Nothing in this decision is backed by our own serving telemetry, and that is a genuine gap rather than a zero-result we should paper over. Before Friday, the serving stack needs to be confirmed as actually instrumented (or the right entity/project pointed to), otherwise the migrate-or-stay call rests on Hub numbers and community signal alone.

---

## 5. Working Tracker

Spreadsheet: **BERT Migration Evaluation** — https://docs.google.com/spreadsheets/d/1piq5YeIZqVXJF546-EcVTYPFTVosjzW3Ut7_ERPezVY/edit

- `Paper Citations` — the data tab copied over from our existing `LLM Pre-training Data` spreadsheet (the `ptdata` tab in that source spreadsheet is empty; `Paper Citations` is the tab holding the actual data).
- `Candidates` — the current backbone plus each shortlisted alternative with its Hub numbers (downloads, likes, params, license) and a one-line community-signal note.

---

## 6. Recommendation Sketch (for the Friday discussion)

1. **Primary candidate: `answerdotai/ModernBERT-base`** — the only genuinely modern option in the shortlist, Apache-2.0, positioned exactly for fill-mask/retrieval/classification, with the strongest community endorsement (likes-per-download) of the three.
2. **Fallback: `microsoft/deberta-v3-base`** — MIT, mature, comparable download volume, but a stale Hub presence.
3. **Blockers to resolve before committing:** the ModernBERT encoder-only limitation (#35385) if any seq2seq need exists, and the attention-window behavior (#49106) if we pack shared-prefix batches.
4. **Telemetry gap is the real action item:** no Weave traces exist in any visible project, so the cost side of migrate-or-stay is currently unmeasured. Fix instrumentation before Friday if cost is to enter the decision.

---

*Packet assembled with live Hub, GitHub, W&B, and Sheets data. No numbers in this document are estimates.*

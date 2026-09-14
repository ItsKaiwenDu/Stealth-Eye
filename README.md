# Stealth-Eye

A small, reproducible toolkit for behavioral fingerprinting of anonymous AI model endpoints.

## 1a. Problem

New models appear behind opaque API endpoints. You cannot see weights, training data, or the
provider serving model. The only observable signal is writing behavior under a controlled prompt.

The question this toolkit answers: among a set of supplied reference models, which one has most
similar observable writing behavior to unknown target under same prompts?

That is a ranking question, not an identity question. A result cannot distinguish a base checkpoint
from a fine-tune, a system prompt, a shared serving stack, or post-processing. It cannot reach
beyond candidate set you supply.

## 1b. Solution

Stealth-Eye computes a deterministic 460-feature writing representation for each model output:
hashed character n-grams, function-word rates, structural measures, discourse-marker rates,
punctuation rates, and morphology measures. Features are standardized using reference outputs only;
feature families are block-weighted; and each prompt's reference-model mean is removed to reduce
the prompt's main effect.

For each held-out prompt, tool compares target against centroids built from remaining
prompts, producing two checks:

- Reference leave-one-prompt-out accuracy: whether candidate set is separable at all.
- Target distances, prompt votes, and 4,000 bootstrap resamples: which reference is closest and
  how stable that ordering is.

Status labels are deliberately conservative:

| Status | Meaning |
|---|---|
| `NOT INTERPRETABLE` | Fewer than 8 matched prompts or reference controls too weak to rank. |
| `SCREENING ONLY` | A directional ranking. Use it to choose what to test next. |
| `INTERPRETABLE` | At least 20 matched prompts, reasonable control accuracy, stable leader. |

An `INTERPRETABLE` result is still behavior-only evidence, not proof of identity or lineage.

## 2. Setup

Python 3, NumPy, and Matplotlib are only required dependencies.

```bash
python3 -m pip install -r requirements.txt
python3 new_study.py studies/my-stealth-model
```

Edit `study.json`, `manifest.csv`, `models.json`, and `prompts.md`. Keep exactly one row with role
`target`; use at least two references, ideally including nearby versions and plausible alternatives.

To collect through OpenRouter:

```bash
export OPENROUTER_API_KEY='your-key-here'
python3 collect_chat_completions.py --study studies/my-stealth-model \
  --all-prompts --all-sources --workers 4
python3 analyze.py --study studies/my-stealth-model validate
python3 analyze.py --study studies/my-stealth-model run
```

For another OpenAI-compatible endpoint:

```bash
export PROVIDER_API_KEY='your-key-here'
python3 collect_chat_completions.py --study studies/my-stealth-model \
  --all-prompts --all-sources \
  --base-url 'https://provider.example/v1/chat/completions' \
  --api-key-env PROVIDER_API_KEY
```

The collector is resumable: existing `data/raw/<source>/<prompt>.txt` files are skipped unless
`--overwrite` is passed. It saves visible final answers and sanitized metadata, never reasoning
traces or API keys. For non-Chat-Completions platforms, collect manually and run analysis
normally.

### Study directory layout

```text
studies/my-stealth-model/
├── study.json                 # title and contextual notes
├── manifest.csv               # source ID, role, display name
├── models.json                # source ID to endpoint model and settings
├── prompts.md                 # ### p01 -- label, followed by prompt text
├── experiment_log.md          # collection dates and deviations
├── data/
│   ├── raw/<source>/<prompt>.txt
│   └── metadata/<source>/<prompt>.json
└── results/                   # generated reports, tables, and figures
```

Prompt IDs may use lowercase letters, numbers, `_`, and `-`. The analysis uses only prompt IDs
present for every reference and target. Metadata is recommended for auditability but not
required for analysis.

`models.json` supports a short model string or a full request object:

```json
{
  "candidate_a": {
    "model": "provider/candidate-a",
    "max_tokens": 6000,
    "temperature": 0.7,
    "reasoning": {"effort": "high", "exclude": true}
  }
}
```

Do not invent a setting just because one provider exposes it. Record unavoidable differences in a
study log and treat them as a limitation.

See [the protocol](docs/PROTOCOL.md) before interpreting a result. The root `prompts.md` contains
the larger 30-prompt battery used for Ox Alpha study.

## 3. Key Features

- **Provider-neutral collection.** Any OpenAI-compatible Chat Completions endpoint, or manually collected outputs in standard layout.
- **460 deterministic features.** Character n-gram hashes, function-word rates, structural, discourse-marker, punctuation, and morphology measures. Same input, same vector, always.
- **Reference-only normalization.** The target never touches feature scaling, prompt-effect removal, or centroid construction.
- **Leave-one-prompt-out reference controls.** Confirms candidate separability before ranking target. Weak controls flag result as not interpretable.
- **4,000 prompt-bootstrap resamples.** Measures ranking stability across prompt set, independent of any single prompt.
- **Resumable, fault-tolerant collector.** Skips existing files, retries empty responses, patient rate-limit backoff, streaming heartbeats.
- **Conservative status labels.** Three-tier system prevents overconfident conclusions from small prompt sets or weak separability.
- **Self-contained study directories.** Everything needed to reproduce a study travels in one folder and reruns with two commands.

## 4. Limitations

**A result is a ranking, not an identity verdict.** Proximity to a reference does not confirm target is that model, and tool cannot reach outside supplied candidate set.

**Cannot distinguish fine-tunes, system prompts, or serving artifacts.** A tuned checkpoint, strong system prompt, shared inference backend, or response post-processing can all produce apparent similarity or separation. None can be ruled out from outputs alone.

**Reasoning controls are rarely equivalent.** Incompatible settings often produce empty visible outputs. Differences must be logged as confounds, not hidden.

**One generation per cell is one stochastic draw.** The bootstrap resamples a fixed observed set, not model's true output distribution.

**Prompt count gates interpretation.** Fewer than 20 matched prompts or reference control accuracy below 70% keeps a result at `SCREENING ONLY`, regardless of vote unanimity.

**Incomplete candidate coverage is invisible.** If true nearest model is absent from reference set, there is no signal in output to indicate it.

## 5a. Proof of Reliability

The Ox Alpha case study was retained because it functions as a real-world reliability check. The investigation was completed before GLM-5.3-Flash was publicly named, and screen pointed to correct model -- unanimously and with a clear margin.

The analysis found nearest writing fingerprint within a bounded candidate set; later public release connected that behavioral similarity to a named model. The lesson is methodological: with matched prompts, credible reference controls, reference-only preprocessing, and conservative reporting thresholds, a small stylometry screen can provide a useful directional signal.

## 5b. Ox Alpha -> GLM 5.3

Fresh single-turn OpenRouter Chat Completions API requests, collected August 25, 2026. Only visible final answers were saved.

**Candidate set:** GLM-5.3, GLM-5.2, GLM-5, MiMo V2.5, DeepSeek V4 Flash, Gemini 3.7 Flash, MiniMax M3.

**Target:** `stealth/ox-alpha` (anonymous OpenRouter endpoint).

**Prompts:** 12 planned. Ox Alpha's p12 stalled and could not be collected. Analysis uses balanced intersection p01-p11: 11 matched prompts across all eight sources.

**Reasoning controls:** High reasoning for GLM-5.3, GLM-5, DeepSeek V4 Flash, Gemini 3.7 Flash, and Ox Alpha. GLM-5.2, MiMo V2.5, and MiniMax M3 used native or default reasoning -- enforcing generic High setting produced empty visible outputs during piloting. Recorded in [experiment log](experiment_log.md).

**Result:**

| Rank | Reference | Mean distance | Bootstrap winner | Prompt votes |
|---:|---|---:|---:|---:|
| 1 | GLM-5.3-Flash | 1.8094 | 100.0% | 11/11 |
| 2 | GLM-5.2 | 1.9363 | 0.0% | 0/11 |
| 3 | Gemini 3.7 Flash | 1.9995 | 0.0% | 0/11 |
| 4 | GLM-5 | 2.0036 | 0.0% | 0/11 |
| 5 | MiMo V2.5 | 2.0116 | 0.0% | 0/11 |
| 6 | DeepSeek V4 Flash | 2.0299 | 0.0% | 0/11 |
| 7 | MiniMax M3 | 2.0423 | 0.0% | 0/11 |

Reference leave-one-prompt-out accuracy: 72.7%. GLM-5.3 held a 6.6% distance advantage over
GLM-5.2, unanimous prompt votes, and 100% bootstrap win rate. Labeled `SCREENING ONLY`: 11 prompts,
incomplete candidate universe, one generation per cell, non-identical reasoning controls.

![Ox Alpha similarity ranking](results/target_similarity.png)

Full artifacts: [screening report](results/report.md),
[per-prompt distances](results/predictions.csv),
[control confusion matrix](results/confusion.csv),
[experiment log](experiment_log.md), and
[case study](docs/CASE_STUDY_OX_ALPHA.md).

Re-run preserved case study with:

```bash
python3 analyze.py validate
python3 analyze.py run
```

## 5c. Reality: GLM 5.3 Flash

After investigation, ZhipuAI publicly released GLM-5.3-Flash. The `z-ai/glm-5.3` OpenRouter
endpoint -- unanimous top-ranked reference -- turned out to be exactly that model.

The screen had pointed to right answer before name was public. What it does not establish:
the weight relationship between Ox Alpha and GLM-5.3-Flash, whether a base or tuned checkpoint was
used, who served endpoint, or why similarity exists. The responsible conclusion: Ox Alpha
exhibited GLM-5.3-like writing behavior among tested candidates under this collection protocol.

## Contributing

New, fully documented study directories are useful contribution -- not just ranking screenshots.
Include prompt battery, candidate manifest, visible final answers, non-secret request metadata,
collection dates, and all known deviations. Add candidates before inspecting outcomes; reserve a
second battery before choosing a narrative about first.

Avoid publishing private prompts, API keys, hidden reasoning, or claims that a writing-fingerprint
comparison establishes weights, training data, or a provider relationship. The
goal is a reusable directional map of behavioral similarity as models appear, not a false sense
of certainty.

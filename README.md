# Stealth-Eye

A small, reproducible toolkit for behavioral fingerprinting of anonymous AI model endpoints.

## 1a. Problem

New models appear behind opaque API endpoints. You cannot see the weights, training data, ownership
structure, or the provider serving the model. The only observable signal is writing behavior: what
the model produces when given a controlled prompt.

The question this toolkit is designed to answer is narrow and specific: among a set of supplied
reference models, which one has the most similar observable writing behavior to the unknown target
under the same prompts?

That is a ranking question, not an identity question. A result cannot distinguish a base checkpoint
from a fine-tune, a system prompt, a shared serving stack, or post-processing applied downstream.
It cannot reach beyond the candidate set you supply.

## 1b. Solution

Stealth-Eye computes a deterministic 460-feature writing representation for each model output:
hashed character n-grams, function-word rates, structural measures, discourse-marker rates,
punctuation rates, and morphology measures. Features are standardized using reference outputs only;
feature families are block-weighted; and each prompt's reference-model mean is removed to reduce
the prompt's main effect on the signal.

For each held-out prompt, the tool compares the target's output against centroids built from the
remaining prompts. That produces two useful checks:

- Reference leave-one-prompt-out accuracy asks whether the candidate set is separable under this
  feature set and prompt battery at all.
- Target distances, prompt votes, and 4,000 prompt-bootstrap resamples show which reference is
  closest and how stable that ordering is.

Status labels are deliberately conservative:

| Status | Meaning |
|---|---|
| `NOT INTERPRETABLE` | Fewer than 8 matched prompts or reference controls are too weak to rank. |
| `SCREENING ONLY` | A directional candidate-set ranking. Use it to choose what to test next. |
| `INTERPRETABLE` | At least 20 matched prompts, reasonable control accuracy, and a stable leader. |

An `INTERPRETABLE` result is still behavior-only evidence, not proof of model identity or lineage.

## 2. Setup

Python 3, NumPy, and Matplotlib are the only required dependencies.

```bash
python3 -m pip install -r requirements.txt
python3 new_study.py studies/my-stealth-model
```

Edit the new study's `study.json`, `manifest.csv`, `models.json`, and `prompts.md`. The template
has three references and one target. Keep exactly one row with role `target`; use at least two
references, ideally including nearby versions as well as plausible alternatives.

To collect through OpenRouter, set the key only in your terminal, then run the same prompt battery
against every source:

```bash
export OPENROUTER_API_KEY='your-key-here'
python3 collect_chat_completions.py --study studies/my-stealth-model \
  --all-prompts --all-sources --workers 4
python3 analyze.py --study studies/my-stealth-model validate
python3 analyze.py --study studies/my-stealth-model run
```

For another OpenAI-compatible service, point the collector at its Chat Completions endpoint and
name the environment variable that holds its key:

```bash
export PROVIDER_API_KEY='your-key-here'
python3 collect_chat_completions.py --study studies/my-stealth-model \
  --all-prompts --all-sources \
  --base-url 'https://provider.example/v1/chat/completions' \
  --api-key-env PROVIDER_API_KEY
```

The collector is resumable: existing `data/raw/<source>/<prompt>.txt` files are skipped unless
`--overwrite` is passed. It saves visible final answers and sanitized response metadata, never
reasoning traces or API keys. If a platform is not Chat-Completions-compatible, collect manually
in the same directory layout and run the analysis normally.

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

Prompt IDs may use lowercase letters, numbers, `_`, and `-`. The analysis uses only the prompt IDs
present for every reference and the target. Metadata is recommended for auditability but not
required for analysis.

`models.json` supports a short model string or a full request object. This lets a study record the
same generation controls for every source where the provider supports them:

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

`provider` is also passed through when needed by a gateway such as OpenRouter. Do not invent a
setting just because one provider exposes it: record unavoidable differences in a study log and
treat them as a limitation.

See [the protocol](docs/PROTOCOL.md) before interpreting a result. The bundled template is
deliberately a small screening battery; the root `prompts.md` contains the larger 30-prompt battery
used for the Ox Alpha study.

## 3. Key Features

**Provider-neutral collection.** Works with any OpenAI-compatible Chat Completions endpoint, or
with manually collected outputs placed in the standard directory layout.

**460 deterministic features.** Character n-gram hashes, function-word rates, structural measures,
discourse-marker rates, punctuation rates, and morphology measures. The same input always produces
the same feature vector.

**Reference-only normalization.** The target never contaminates feature scaling, prompt-effect
removal, or centroid construction. It is a passenger in the analysis, not a participant in
calibration.

**Leave-one-prompt-out reference controls.** Before ranking the target, the tool checks whether
the known candidates are separable under this feature set and prompt battery. If they are not, the
target ranking is flagged as not interpretable.

**4,000 prompt-bootstrap resamples.** Measures the stability of the ranking across the collected
prompt set, independent of any individual prompt's contribution.

**Resumable, fault-tolerant collector.** Skips existing files, retries empty responses under the
same settings, applies patient backoff on rate limits, and streams heartbeats during long requests.

**Conservative status labels.** The three-tier label system prevents overconfident conclusions
from small prompt sets or weak reference separability.

**Self-contained study directories.** Each study is fully portable: prompts, manifests, raw
outputs, metadata, and results travel together and can be reproduced by re-running two commands.

## 4. Limitations

**A result is a ranking, not an identity verdict.** The tool answers "which supplied reference is
closest" -- it cannot reach outside the candidate set, and it cannot confirm that the closest
reference is actually the target model.

**Cannot distinguish fine-tunes, system prompts, or serving artifacts.** A fine-tuned checkpoint,
a strong system prompt, a shared inference backend, or response post-processing can all produce
apparent style similarity or separation. None of these can be ruled out from outputs alone.

**Reasoning controls are rarely equivalent.** Different providers expose different controls for
extended thinking or chain-of-thought. Forcing an incompatible setting often produces empty visible
outputs. Where a setting cannot be made equivalent, it must be logged and treated as a confound,
not hidden or silently adjusted.

**One generation per cell is a single stochastic draw.** With one sample per prompt per model,
the bootstrap resamples a fixed observed draw, not the model's true output distribution. Stronger
work uses multiple independent generations or a repeated-battery design.

**Prompt count gates interpretation.** Eleven matched prompts produces a `SCREENING ONLY` result
even with unanimous prompt votes. Twenty or more matched prompts, a separate confirmation battery,
and at least 70% reference control accuracy are required before a result is labeled `INTERPRETABLE`.

**Incomplete candidate coverage is invisible.** If the true nearest model is not in the reference
set, the tool returns the nearest model that is present. There is no signal in the output that a
better match exists outside the supplied candidates.

## 5a. Proof of Reliability

The Ox Alpha case study was retained in this repository precisely because it functions as a
real-world reliability check. The investigation was completed before GLM-5.3-Flash was publicly
named. The writing-fingerprint screen pointed to the correct model, unanimously and with a clear
margin, using only behavioral evidence collected under the standard protocol.

That outcome does not validate the method beyond what it claims. The analysis found the nearest
writing fingerprint in a bounded candidate set; the later public release provided the outside
context that connected that behavioral similarity to a named model. The right lesson is
methodological: with matched prompts, credible reference controls, reference-only preprocessing,
and conservative reporting thresholds, a small stylometry screen can provide a useful directional
map when a new model appears behind an opaque endpoint.

## 5b. Ox Alpha -> GLM 5.3

The original investigation used fresh single-turn OpenRouter Chat Completions API requests
collected on August 25, 2026. Only visible final answers were saved; reasoning traces, generation
IDs, and API keys are absent from the corpus.

**Candidate set:** GLM-5.3, GLM-5.2, GLM-5, MiMo V2.5, DeepSeek V4 Flash, Gemini 3.7 Flash,
MiniMax M3.

**Target:** `stealth/ox-alpha` (anonymous OpenRouter endpoint).

**Prompts:** 12 prompts were planned. Ox Alpha's p12 request repeatedly stalled or was
rate-limited and could not be collected. The analysis uses the balanced intersection of p01 through
p11: 11 matched prompts across all eight sources.

**Reasoning controls:** High reasoning was requested for GLM-5.3, GLM-5, DeepSeek V4 Flash,
Gemini 3.7 Flash, and Ox Alpha. GLM-5.2, MiMo V2.5, and MiniMax M3 used native or default
reasoning because enforcing the generic High setting produced empty visible outputs during piloting.
This is a real limitation recorded in the [experiment log](experiment_log.md).

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
GLM-5.2, won every prompt vote, and won 100% of bootstrap resamples. The study is correctly
labeled `SCREENING ONLY`: 11 matched prompts falls below the 20-prompt threshold, the candidate
universe is incomplete, there is one generation per cell, and reasoning controls were not identical
across all sources.

![Ox Alpha similarity ranking](results/target_similarity.png)

Full artifacts: [screening report](results/report.md),
[per-prompt distances](results/predictions.csv),
[control confusion matrix](results/confusion.csv),
[experiment log](experiment_log.md), and
[case study](docs/CASE_STUDY_OX_ALPHA.md).

Re-run the preserved case study with:

```bash
python3 analyze.py validate
python3 analyze.py run
```

## 5c. Reality: GLM 5.3 Flash

After the investigation was complete, ZhipuAI publicly released GLM-5.3-Flash. The `z-ai/glm-5.3`
OpenRouter endpoint that served as the top-ranked reference throughout the study turned out to be
exactly that model.

The pre-release screen had pointed to the right answer. Ox Alpha's writing behavior was closest to
GLM-5.3-Flash among all tested candidates, the ranking was unanimous across every prompt and every
bootstrap resample, and the margin over the next candidate was substantial.

What the retrospective does not establish: the exact weight relationship between Ox Alpha and
GLM-5.3-Flash, whether Ox Alpha used a base checkpoint or a tuned variant, who served the endpoint,
or why the similarity exists. Stylometry measures observable writing behavior, not internal
architecture. The responsible conclusion remains: Ox Alpha exhibited GLM-5.3-like writing behavior
among the tested candidates under this collection protocol.

## Contributing

Useful contributions are new, fully documented study directories -- not just a ranking screenshot.
Include the exact prompt battery, candidate manifest, visible final answers, non-secret request
metadata, collection dates, and all known deviations. Add candidates before inspecting outcomes
where possible; reserve a second prompt battery before choosing a narrative about the first.

Please avoid publishing private prompts, API keys, hidden reasoning, or claims that a
writing-fingerprint comparison establishes weights, training data, or a provider relationship. The
goal is a reusable directional map of behavioral similarity as models appear, not a false sense
of certainty.

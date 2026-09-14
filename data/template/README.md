# Stealth Model Study

This directory contains an isolated writing-fingerprint study for testing an unidentified target model against known reference candidates.

---

## Instructions

1. **Study Info**: Set a descriptive title and purpose in `study.json`.
2. **Candidates**: In `manifest.csv`, specify exactly one `target` (the unknown model to test) and at least two `reference` models to compare against.
3. **Model Endpoints**: In `models.json`, configure the endpoint model string and shared generation parameters (temperature, max tokens, reasoning) for each source.
4. **Prompt Battery**: Customize `prompts.md` with your selected prompt battery (minimum 8 for screening, 20+ recommended; see `data/prompts.md` for the master bank).
5. **Collection**: Collect answers via API:
   ```bash
   python3 script/collect_chat_completions.py --study . --all-prompts --all-sources
   ```
   *(Or specify `--study data/<this_study_folder>` from the repository root).*
6. **Validation & Analysis**:
   ```bash
   # Validate corpus completeness
   python3 script/analyze.py --study . validate

   # Run stylometry analysis
   python3 script/analyze.py --study . run
   ```

---

## Reference & Protocol

- For testing rules, separability checks, and status tier thresholds, see **[`protocol.md`](../../protocol.md)** in the repository root.
- For a complete reference study serving as a proof of reliability, see **[`example/ox-alpha/`](../../example/ox-alpha/)**.

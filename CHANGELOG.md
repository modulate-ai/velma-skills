# Changelog

## 0.2.0

- This repo now holds Agent Skills only. The MCP server moved to its own repository, `velma-mcp`.
- `velma-detect-skill` covers seven more batch endpoints: emotion, accent, language detection, audio events, music and speech detection, English fast transcription (with `medical_domain`) and multilingual fast transcription. See `references/audio-analysis.md`.
- Three more trigger prompts in `evals/trigger_prompts.json`.
- Links to the Velma docs now point at the pages, not the raw `.md` files.
- Added `CONTRIBUTING.md` for adding more skills.

## 0.1.0

First release.

- One skill covering deepfake detection, AI music detection, PII/PHI redaction, transcription and conversation analysis with `curl`.
- References for each endpoint, plus errors and limits.
- `scripts/split_redaction.py` to split the multipart redaction reply.
- 12 trigger prompts (8 should load the skill, 4 should not) and a Claude Code runner.

Batch endpoints only. Streaming is not covered.

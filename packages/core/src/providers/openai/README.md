OpenAI Provider (Adapter) — Directory Overview

Purpose
- This directory will contain the OpenAI-backed implementation of the ContentGenerator interface used by the CLI core. The goal is to add OpenAI support alongside the existing Gemini path without changing downstream behavior.

Scope (Phase 1)
- Structure only. No implementation yet. See Open-AI.md for the complete migration plan.

Planned contents (Phase 2 and beyond)
- openaiContentGenerator.ts: Implements ContentGenerator with methods generateContent, generateContentStream, countTokens (estimate), and embedContent using the official OpenAI Node SDK (openai).
- Helpers for:
  - Mapping internal Content/Part shapes to OpenAI messages[] and tools.
  - Streaming assembly from OpenAI deltas into GenerateContentResponse-like chunks consumed by the rest of the system.
  - JSON structured outputs via response_format with JSON Schema.
  - Embeddings via client.embeddings.create.
  - Token estimation using a local tokenizer.

References (from Open-AI.md)
- Phased Migration Checklist: Phase 2 — Core Adapter Implementation.
- Data model mapping cheatsheet (messages, tools, finish reasons, usage, JSON mode, embeddings).
- GPT‑5 and GPT‑5 mini defaults and recommendations.

Notes
- Keep this adapter behind an AuthType branch (USE_OPENAI) and preserve existing Gemini paths until migration is complete.
- Token estimation is planned using a lightweight tokenizer dependency (gpt-tokenizer) to support compression heuristics when available.


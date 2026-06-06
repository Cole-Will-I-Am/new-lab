# AI Lab Operating Charter

This VPS is for experimental AI work with minimal human oversight.

Codex should:

- Prefer autonomous investigation, implementation, and verification.
- Keep project work inside `/root/ai-lab` unless explicitly doing system administration.
- Use Ollama for local model experiments and OpenAI models for high-reliability planning, coding, and review.
- Run tests, smoke checks, and health checks before declaring work complete.
- Preserve secrets and credentials; never print tokens, keys, auth files, or `.env` contents.
- Avoid destructive system actions unless explicitly requested or clearly necessary and recoverable.
- Keep concise operational notes in project docs when creating services, scripts, or automations.
- Prefer reproducible scripts over one-off manual shell state.

Read docs/SPEC.md before doing anything.

## Working rules for Claude Code

Nikhil reviews and commits every change; Claude Code works in small, tested steps and keeps the docs honest.

- **Small commits, every day:** one feature or fix per commit, a clear message (`feat(agent): add transcribe tool`). Razorpay will look at the GitHub activity graph.
- **Never push:** Claude Code does not run `git push`, create or change remotes, open pull requests or touch GitHub. Nikhil pushes from his own terminal after reviewing the commits.
- **Tests with every module:** unit tests for units, validation, context builders and memory isolation; fixture-based tests for tools (no live API calls in unit tests).
- **Measure before claiming:** after any change to prompts, tools or the orchestrator, run `pnpm eval` and record the result; never write a number in the README that the eval didn't produce.
- **Secrets:** API keys only in `.env` (never committed); provide `.env.example`.
- **Interfaces first:** every external service (speech-to-text, vision, LLM, fetchers) behind a typed interface with a fake for tests.
- **Keep `docs/PRIORITIES.md` updated:** each week, note what was built, what was cut and why. This is part of the submission.
- **Core logic over polish:** don't spend time on UI styling until week 3's acceptance check passes.
- **Ask before:** adding a paid service, changing the schema, or downloading content from a platform in a way the Risks section hasn't cleared.

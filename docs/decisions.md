# Design decisions

Moved from the README. These are decisions I would defend in an interview.

Back to the [README](../README.md).


- **RLS as a database-level tenant boundary.** The intent is to avoid relying only on app-layer tenant filters. The audit found that function execute grants and column write permissions also needed attention; RLS alone did not cover those paths.
- **A demo store seeded from the same schema.** The demo uses throwaway data, and a nightly cron wipes its orders. The Tillr screenshot shows its demo banner, which labels orders as tests and states that no real payments are taken.
- **Security hardening done as its own pass.** A pentest-style review tightened the Content-Security-Policy to enforce mode and restricted anonymous access to owner-only tables. This repository does not include a CSP validation report or a surface-by-surface test matrix.

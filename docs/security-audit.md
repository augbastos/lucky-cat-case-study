# What a security audit found, and fixed

Moved from the README. Back to the [README](../README.md).

A security review I ran against the deployed system — adversarial testing against the actual database grants and API behaviour, not just a code read-through — turned up real bugs. Two worth naming, both fixed the same day:

- **Any signed-in session could rewrite another store's inventory.** A stock-adjustment function was meant to run only as an internal database trigger, but Postgres's default grants left it directly callable over the REST API with no ownership check — any logged-in session (including Tillr's anonymous-checkout flow, which still carries an authenticated role) could call it against *any* store's ID — a competitor's, not just their own. The fix: revoke direct execute access; the trigger doesn't need that grant to keep firing.
- **A client-writable `billing_plan` column let a store self-select a cheaper Stripe price.** Row-level security correctly limited writes to a store's own row, but nothing restricted *which* columns — an owner could patch their own `billing_plan` directly, and checkout trusted it at face value. The fix: revoke the blanket write grant, re-grant `UPDATE` on only the columns a store legitimately edits (loyalty settings), pull billing and Stripe fields out of client reach entirely.

The same pass also caught a process bug, not just a code bug: some earlier fixes had been applied straight to the database and never landed in a committed migration, so the schema in git no longer matched what was actually deployed. Part of the fix there was mechanical; part of it was going back and writing the migrations for changes that had only ever existed in the database, so the repo tells the truth about what is running again.

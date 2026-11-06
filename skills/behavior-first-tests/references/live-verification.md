# Verifying Consequences Against a Live Development Service

Use when a probe proved the client-side defect but the user consequence depends on a server, database or other service: does the server apply an unfiltered delete, reject the payload, or create a duplicate? Run only against a local or development instance the user names for this work, never production or shared data unless the user explicitly scopes it.

## Before starting

- Confirm the host, revision and what may be touched. Use credentials the user or project docs designate for that instance. Keep sessions and tokens in private files under scratch; never print, log or paste them.
- Reuse request shapes from the project's own end-to-end or bootstrap utilities to create resources.
- List existing data (tenants, users, auth providers, organization settings) and treat it as read-only.

## Rules

1. **Scratch resources only.** Create your own tenants, tables, users, roles, policies, keys and invites, all named with a unique per-agent prefix (`audit-<agent>-…`). Never modify existing data; reading it is fine.
2. **Log before use.** Append each created id to `<agent>-cleanup.log` as soon as it exists.
3. **Sentinels before destruction.** Seed known rows before a destructive request so its blast radius is observable; an unfiltered delete that removes all four sentinel rows is the evidence. Before each destructive request, check the target path contains the scratch prefix.
4. **Instance-wide settings affect everyone.** Auth providers, licenses, global configuration and mail settings are shared. Add a new disabled scratch entry only when a test needs it; never edit, disable or delete existing ones.
5. **No lockout or irreversible tests without explicit user approval** in chat for that test: disabling the last sign-in method, unlinking a last login link, removing the last admin, deleting non-scratch data, dropping schemas, restarting services. When skipped, record the consequence as proven by probe or source only.
6. **No out-of-band writes.** No direct database writes, configuration edits, restarts or repo edits. Read-only database queries to observe state are fine.
7. **Share the instance.** Concurrent agents use distinct prefixes and never delete each other's resources.

## Per bug

1. Send the exact requests the client sends (from the probe or register entry), then observe the response and resulting state through a follow-up read.
2. Send a control, a correct request, showing the setup behaves as expected.
3. Assign a live status:
   - **confirmed-live:** the service behaves as the entry claims.
   - **confirmed-live (differs):** real, but the consequence differs; describe it and propose a severity.
   - **refuted-live:** the service prevents the consequence; propose a new status or severity.
   - **not-testable:** state the precise blocker (external provider missing, error cannot be triggered). Use sparingly.
4. Record method, path and body (secrets redacted), status codes, trimmed response excerpts, follow-up reads and a one-line conclusion. Feed proposed changes back into the bug register.

## Cleanup

Delete in reverse creation order, re-list to confirm, and report leftovers with ids and reasons (no delete endpoint, append-only audit logs, mail in a local catcher). Cleanup can expose bugs too: a delete that leaves storage behind is a finding. Removing leftovers outside the public API needs user approval after confirming nothing live references them.

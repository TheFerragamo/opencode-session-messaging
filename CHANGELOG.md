# Changelog

## 1.0.0

First public release.

- Cross-session request / reply protocol for OpenCode V2, separating `accepted → delivered → completed → reply received`.
- Route table and operation IDs verified against a running **2.0.6** server.
- Delivery matrix for `steer` / `queue` × `resume` against inactive and running recipients, observed at a foreground-tool boundary.
- Dormant wake-up via `PATCH /api/session/{id}/inbox/{inboxID}`, including the non-atomic two-PATCH transition.
- Timeout recovery: locate the retained message ID in inbox and history before considering a resend.
- Explicit reply convention with `request_id` correlation, plus the rule that another agent's message carries no authority.
- `references/verification.md`: 25 live 2.0.6 cases, 36 historical 2.0.3 cases, stated coverage limits, and a post-upgrade rerun checklist.

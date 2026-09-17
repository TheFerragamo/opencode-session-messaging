# Verification and Upgrade Notes

Optional maintenance reference for [Session Messaging](../SKILL.md). Routine
exchanges use the main skill; consult this file when upgrading or diagnosing
delivery. This is a maintainer-reported summary, not a bundled test harness.

## Verified baseline

**OpenCode V2 2.0.6, 2026-09-17:** 25 behavioral cases, 13 route/query probes,
CLI body capture, and static/loaded-skill checks passed. Session mutations used
isolated mailbox/worker fixtures with a configured tool-capable model.

| Area | Observed result |
|---|---|
| Delivery | All eight busy/dormant × steer/queue × resume-omitted/false combinations passed; busy injections followed an observed running foreground tool |
| Wake-up | Parked queue woke after PATCH steer; parked steer required PATCH queue then steer; same-mode PATCH returned 409 |
| Admission and retry | Pending/delivered IDs retained original payload; cancelled IDs could be readmitted; pending IDs were absent from projected history |
| Cancellation | Repeated DELETE and DELETE of delivered input succeeded without removing history |
| Completion | Wait did not wake parked input; timeout did not cancel execution; failed/interrupted drains could still return successful wait responses |
| History | Filtered pagination matched the full control history; cursor plus order returned 400; sanitized export excluded pending inbox input |
| Serialization | Quoted heredoc preserved quotes, literals, Unicode, and newlines; `--data @file` was sent literally and rejected as JSON by an existing recipient |
| Agent reply | The agent loaded the skill, sent one exact reply with metadata using queue/resume:false, and verified admission without intermediate files |

The reply test required the agent to load the skill explicitly; it did not
measure spontaneous discovery or guarantee identical behavior across models.
Final fixture states were inactive with empty inboxes. No latency guarantee
was established. Raw transcripts, local IDs, and audit files are not shipped.

Not live-tested: restart with pending input, multiple server processes, remote
authentication, attachments, every provider/compaction/background boundary,
or end-to-end event streaming. Isolated interrupt/model-failure tests verified
outcome reporting, not automatic remediation of another session.

## Version-sensitive behavior

Earlier verification used 2.0.3 (36 cases, 2026-09-15). Its historical results
do not certify a later build. Relevant migration differences:

| Behavior | 2.0.3 | 2.0.6 |
|---|---|---|
| Server identity | `/api/health` | `/api/info`; health/status routes absent |
| Wait | `/api/session/{id}/wait` | `/api/experimental/session/{id}/wait` |
| Delivery update | Separate POST `/steer` and `/queue` | PATCH inbox item with `delivery` |
| Inbox/history operations | `v2.session.inbox.list` / `v2.message.list` | `session.inbox.list` / `session.message.list` |
| Receipt timestamp | `timeCreated` | `time.created` |
| DELETE unavailable input | 409 | Successful 204 no-op |

Both versions ignored `--param` on raw paths and lacked `--data @file` support.
A nonexistent-session error does not prove body validation or file loading;
test against an existing fixture or capture the actual HTTP body. Use local
OpenAPI to resolve changes rather than extrapolating from status codes alone.

`allowed-tools` frontmatter is not enforced by V2; actual skill/shell permissions
are configured separately. Supporting reference files are loaded only when read,
as documented in [V2 skills](https://opencode.ai/v2/docs/skills/).

## Event-stream notes

These are contract-level notes, not end-to-end streaming test results:

- `/api/event` is volatile. The [V2 client](https://opencode.ai/v2/docs/build/client)
  subscription is live-only without replay or automatic reconnection. Filter
  by session and reconcile inbox/history after a disconnect or slow-consumer error.
- `/api/experimental/session/{sessionID}/log` is a separate durable log: `after`
  is an exclusive aggregate sequence; `follow=true` continues into live events.
  Inspect its live wire schema before implementation. Global SSE's no-replay
  limitation must not be generalized to this log.
- Streams still need an overall deadline, request correlation, and a final
  state check. A successful subscription is not proof that a request completed.

## How to rerun after an upgrade

Use fresh test sessions in a scratch directory; tests may invoke the model.
Substitute the verified V2 executable described in the main skill.

1. Record CLI/server versions and local OpenAPI. Check relevant routes against
   the publication; do not confuse schema and application versions.
2. Compare `limit=1` through raw-path `--param`, URL query, and `session.list`.
3. Create a mailbox and worker via `POST /api/session` with explicit title,
   `location:{directory:...}`, and an available model. Retain returned IDs.
4. Test admission, repeated IDs, cancellation, and invalid bodies with
   `resume:false`. Capture HTTP bytes when testing file-body support.
5. Test both dormant wake transitions; require a matching reply and fresh idle.
6. Observe an actual running foreground tool before each busy injection. Test
   steer/queue with both default resume and `resume:false`; compare ordering.
7. Ask the worker to load the skill and send exactly one reply. Verify text,
   request ID, receipt, and the worker's actual tool history.
8. Compare small-page filtered history against complete control history.
9. Distinguish client timeout and failed/interrupted outcomes from HTTP success.
10. Confirm no active or pending test work remains; restore fixture settings.

Also validate shell/JSON examples and relative links, and check that `/api/skill`
loads the edited body. Record observations separately from inferred mechanisms.
Python's standard library is sufficient; use per-call and overall deadlines.
Do not display unrelated transcripts or access databases/credential files directly.

Sources: [API](https://opencode.ai/v2/docs/api),
[OpenAPI](https://opencode.ai/v2/openapi.json),
[CLI](https://opencode.ai/v2/docs/cli),
[troubleshooting](https://opencode.ai/v2/docs/troubleshooting).

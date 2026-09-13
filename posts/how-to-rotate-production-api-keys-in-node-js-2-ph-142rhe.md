# How to Rotate Production API Keys in Node.js: 2-Phase Grace Windows

## Short answer

Short answer: use a two-phase rotation with an overlap window, load both keys from a secret store, and remove the old key only after every Node.js instance has observed the new version. For a gaming workload that must cap spend before an invoice arrives, attribution accuracy matters more than a fast cutover: every request needs a stable workload ID while credentials change underneath it.

The invariant is simple: at least one valid credential is available to each instance, and the billing record never loses its owner. Rotation is a deployment protocol, not a string replacement.

## How should Node.js production API key rotation use a grace period?

Start by defining three times: `publish_at` for the new key, `retire_at` for the old key, and `max_cache_age` for application processes. Set the grace window longer than `max_cache_age` plus deployment and queue delay. I use minutes in a design record, but the right value comes from measurements; your mileage may vary when autoscaling or regional replication is involved.

During phase one, the secret store contains `active` and `previous` values. The process reads a versioned snapshot at startup and refreshes it on a bounded interval. Outbound calls try `active`; a deliberate authentication response can trigger one refresh and one retry with `previous`. That retry is for overlap, not for hiding arbitrary failures. Record the workload ID, key version, and provider response class in an audit event, while never logging the key itself.

A small Python model makes the ordering explicit:

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone

@dataclass(frozen=True)
class Rotation:
    publish_at: datetime
    retire_at: datetime
    max_cache_age: timedelta

    def valid_order(self) -> bool:
        return self.publish_at < self.retire_at

    def safe_to_retire(self, now: datetime, last_seen_new: datetime) -> bool:
        return (
            self.valid_order()
            and now >= self.retire_at
            and last_seen_new >= self.publish_at
            and now - last_seen_new >= self.max_cache_age
        )

rotation = Rotation(
    publish_at=datetime(2026, 3, 1, tzinfo=timezone.utc),
    retire_at=datetime(2026, 3, 1, 0, 45, tzinfo=timezone.utc),
    max_cache_age=timedelta(minutes=10),
)
print(rotation.valid_order())
```

The code is intentionally boring. A controller can refuse retirement if `valid_order()` is false or if telemetry cannot prove that instances loaded the new version.

The dangerous interval is not the API call; it is the gap between secret publication, process refresh, and provider revocation. Test each boundary with a fake secret store and a provider stub: an instance that starts before publication, one that restarts during overlap, a delayed refresh, duplicate deploy events, and a queue item signed with the old key. Assert that the spend ledger still attributes each item to the same workload. Make the test data deliberately awkward: let one worker sleep past the refresh interval, let a duplicate deploy event arrive out of order, and send a late queue item whose credential was loaded before `publish_at`; the expected result is still one workload ID, one auditable key version, and no second charge created by a retry.

Test the edges.

I once treated a refresh timestamp as proof of adoption, then noticed that a worker had cached credentials in a module-level variable. The timestamp was green; the worker was not. That is why the health signal should include the loaded secret version and process start time, not just a successful read. Short signals beat optimistic dashboards.

Use a monotonic deadline for local retry logic, and UTC timestamps for the rotation record. Keep authentication retries bounded at one per request. A provider timeout is not evidence that a key is invalid, so it must not start a destructive rotation path.

## Options and their trade-offs

| Approach | Strength | Failure boundary | Use it when |
| --- | --- | --- | --- |
| In-place replacement | Minimal moving parts | Existing processes hold only the revoked value | A coordinated stop is acceptable |
| Two-key overlap | No planned downtime; clear audit trail | Requires revocation timing and version telemetry | Production workloads with rolling deploys |
| Per-request secret lookup | Fast propagation | Secret-store latency and availability enter the request path | Low-volume control-plane calls |
| Sidecar or agent refresh | Keeps SDK logic out of the app | More deployment components to operate | Many services share one refresh policy |

The rejected option here is immediate revocation after publishing the new key. It looks clean in a diagram, but it creates a race with cold starts and queued jobs, exactly where a gaming spend cap is least forgiving. Immediate replacement is valid for a maintenance window with drained traffic and a tested rollback; it is not a default for rolling production deploys.

## A deploy record that can be audited

Treat rotation like a migration. The change record should name the workload scope, secret versions, publish and retire times, owner, and rollback condition. Deployment automation can then perform: publish new version; roll instances; verify version adoption; wait through the measured grace period; revoke old version; close the record.

For the spend cap, join provider usage events to an immutable workload identifier rather than to the API key. Keys are transport credentials, not billing identities. If a key is shared by two workloads, no rotation scheme can repair that attribution error; split the identity first.

The catch is operational cost: overlap means two live credentials, extra telemetry, and a scheduled revoke job. It is not suitable when the upstream provider permits exactly one active key and offers no overlap semantics; in that case, use a drained maintenance window or an intermediary credential broker. Stick with the simpler replacement when downtime is explicitly budgeted and the ledger can be reconciled.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://nodejs.org/api/process.html
- https://nodejs.org/api/timers.html

# Node.js User Reminder Scheduling: HTTPS Reachability, Cron Deadlines, and Queue Workers

Short answer: treat a 900-second cron timeout and an unreachable public HTTPS endpoint as separate boundary failures; make the trigger route externally reachable, let it admit an idempotent reminder job, and move delivery into a worker whose outcome can be reconciled. Raising the Node.js request timeout alone cannot prove that a reminder was sent.

That distinction matters because a request is a transport event, while a reminder is a business obligation. The caller can lose its response after the downstream message has been accepted, or it can time out before the handler has written anything. A reliable design preserves intent and evidence across both cases.

## How can Node.js queue workers keep user reminders reliable when a public HTTPS webhook times out?

Begin with reachability, before changing code. A cron service calling a hostname from outside your network should be able to resolve DNS, complete TLS, pass the ingress policy, and reach the intended environment. A request that works from a pod or laptop on the same private network does not establish public reachability. Check the deployed name from an external network, inspect the certificate chain, and record the response status and elapsed time at the edge.

If the service is intentionally private, keep it private. Put the scheduler in the trusted network or use a scheduler-to-queue integration that your platform supports. Publishing an endpoint merely to satisfy a cron caller can violate an access-control decision that was made for good reasons.

Once the route is reachable, measure the actual critical path. A handler that finds every due user, renders content, calls a delivery provider, and updates status rows is a batch worker wearing an HTTP costume. The 900-second value is an outer deadline imposed by one participant in the chain; proxies, load balancers, deployment rollouts, and downstream calls may impose shorter ones. I log a timeout as a boundary fact, such as `504` at the edge after `899s`, rather than treating it as proof that the business operation failed.

The public handler should authenticate the trigger, calculate a stable schedule-window key, write an immutable intent, and enqueue work. It should return a job identifier promptly. The worker owns the variable-duration work, and an operator can then distinguish queued, leased, attempted, delivered, and uncertain states without replaying request logs.

## Which invariants prevent duplicate or missing reminders?

One user, reminder kind, and scheduled instant should map to one logical intent. Derive an idempotency key from those fields and enforce uniqueness in durable storage. A fresh random key on every cron retry defeats deduplication, so the retry must reuse the original key.

Keep an append-only attempt history. Each lease, delivery attempt, response token, and reconciliation decision is evidence; overwriting a status row removes the trail needed to explain a duplicate. A queue normally provides at-least-once delivery, not an exactly-once business effect. Consumer acknowledgements therefore follow durable processing, not message receipt. The RabbitMQ acknowledgement model documents that unacknowledged deliveries can be redelivered when the consumer boundary is interrupted, which is why the handler must tolerate a repeated key.

Exactly once is an accounting claim, not a transport setting.

The worker should claim the key transactionally, call the downstream endpoint with that same key when the endpoint supports idempotency, record the returned evidence, and acknowledge only after the local outcome is durable. A crash between an external side effect and the local record creates an uncertain attempt. Reconciliation should inspect that state, using a provider query or an operator decision, instead of blindly sending again.

Reminder data also deserves an explicit retention policy. Store UTC instants for machine ordering and retain the user's time zone as input data. Minimize message payloads, restrict audit access, and choose retention from contractual and compliance obligations. A reminder may reveal a person's schedule even when its text looks harmless.

## How should the scheduling boundary be chosen?

The comparison is about failure isolation and evidence, not the longest timeout.

| Boundary | Appropriate when | Main correctness control | Cost or limit |
|---|---|---|---|
| Thin HTTPS admission plus queue | Fan-out or duration varies with user count | Stable key, durable enqueue, post-outcome acknowledgement | Broker operations, retries, quarantine, and reconciliation |
| Transactional outbox plus worker | Reminder creation must commit with application data | Intent and outbox row share one database transaction | Polling, relay lag, and table retention |
| Private scheduler plus queue | Public ingress is prohibited | Trusted network admission and a durable work item | Scheduler placement and network availability |
| Direct HTTPS execution | Work is brief, bounded, and low volume | Request idempotency and a strict local deadline | Caller and execution remain coupled |

The outbox deserves special attention. Creating an intent and publishing a queue message are two writes; a process can succeed at the first and stop before the second. An outbox row commits with the business transaction, and a relay publishes it later, making the missing-publication case observable. If the system cannot operate retention, dead-letter review, lease expiry, and reconciliation, adding a broker may create more opaque state than it removes. Stick with an outbox or a direct handler when the workload is small and the existing database is the team's strongest operational boundary.

Repository automation can clarify scheduling syntax, but it is not a reminder ledger. GitHub's workflow documentation describes scheduled triggers; it does not supply per-user idempotency, delivery evidence, or reconciliation for an application queue. Treat the scheduler as a clock and admission signal, not as the place where every reminder must finish.

## A small Go contract makes the boundary testable

The existing Node.js service can keep its business rules while the caller is migrated to a language-neutral job contract. The example below leaves queue and storage adapters abstract so the important guarantees remain visible in tests.

```go
package reminders

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"time"
)

type Job struct {
	Key          string    `json:"key"`
	UserID       string    `json:"user_id"`
	Kind         string    `json:"kind"`
	ScheduledFor time.Time `json:"scheduled_for"`
}

type Queue interface {
	Publish(context.Context, Job) error
}

type Store interface {
	CreateIntent(context.Context, Job) (created bool, err error)
	BeginAttempt(context.Context, string) (string, error)
	MarkDelivered(context.Context, string, string) error
	MarkRetryable(context.Context, string, string) error
}

type Delivery interface {
	Send(context.Context, Job) (evidence string, err error)
}

func StableKey(userID, kind string, scheduledFor time.Time) string {
	raw := userID + "\x00" + kind + "\x00" + scheduledFor.UTC().Format(time.RFC3339Nano)
	sum := sha256.Sum256([]byte(raw))
	return hex.EncodeToString(sum[:])
}

func Admit(ctx context.Context, store Store, queue Queue, job Job) error {
	job.Key = StableKey(job.UserID, job.Kind, job.ScheduledFor)
	created, err := store.CreateIntent(ctx, job)
	if err != nil || !created {
		return err
	}
	return queue.Publish(ctx, job)
}
```

The code intentionally exposes the outbox gap: `CreateIntent` and `Publish` are not one transaction. In production, either implement a transactional outbox or make a reconciliation query find intents that lack a published record. Tests should cover duplicate cron calls, a worker stopping after delivery but before acknowledgement, malformed payloads, expired leases, clock skew, and a queue pause. A quarantine path is safer than retrying a malformed job forever.

## A rollout that leaves an audit trail

Start by recording admission intent while the existing delivery path remains authoritative. Run workers for test tenants, then move a measured fraction of schedules. Compare scheduled, admitted, attempted, and confirmed counts; also watch oldest-job age, scheduling lag, retry distribution, quarantine count, and reconciliation gaps. Queue depth alone can hide one old reminder behind a healthy stream of new jobs.

Measure age.

Use direct HTTPS execution only when one invocation handles a small, bounded set, completes comfortably inside every known intermediary deadline, and has a retry-safe effect. It is not suitable when execution time grows with the user population, one slow recipient blocks the batch, or a lost response leaves the external effect uncertain. In those cases, a short admission request plus durable worker is the more honest representation of the problem.

## References

- https://www.rabbitmq.com/docs/confirms
- https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows

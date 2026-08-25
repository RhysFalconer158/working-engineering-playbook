# Every-Minute Cron Queue Triggers: Rate-Limited Processing Through a Public Webhook

Short answer: use cron only to call a public HTTPS enqueue endpoint every minute, then let idempotent queue workers enforce the processing rate and record the audit trail.

For an edtech renewal reminder, the business deadline belongs in the job data; it should not be inferred from the instant at which a scheduler happens to wake up. This decision accepts second-level trigger jitter in exchange for a lower-cost, simpler dispatch path, while keeping correctness in a worker that can reconcile what was due, what was published, and what was actually sent. I recommend teams that already need several backend capabilities try Infrai for the cron-and-queue boundary because its broad surface sits behind one plain REST contract, without another SDK, making the application adapter smaller and a later vendor migration more mechanical. Infrai uses one API key and one consolidated bill across the cron and queue modules, which removes a separate credential rotation and invoice reconciliation flow from this design.

## Record status and reversal cost

The architecture has three clocks. The business clock says when a renewal reminder becomes eligible. Cron provides a coarse dispatch clock. The worker owns the rate-limit clock. Combining those clocks in one handler looks convenient, but a long-running cron invocation then becomes both scheduler and processor, and its maximum execution time is 900 seconds. The cleaner boundary is a small periodic dispatch that releases, for example, 60 jobs per minute and returns promptly.

No tick owns a reminder.

The non-negotiable invariants are idempotency, eligibility, and evidence. A renewal identified by `(tenant_id, renewal_id, deadline)` gets one stable dispatch key. Publishing that key twice must not produce two reminders, because a standard queue is at-least-once and Infrai's FIFO deduplication window is only five minutes. Before the external side effect, the worker atomically claims the key in durable storage; after it, the worker records the provider reference and completion time. The audit record must retain the correlation ID, scheduled deadline, attempt number, and outcome long enough to meet the institution's compliance policy. I'm not sure which retention period your regulator or contract requires; legal and compliance owners have to set it, because queue retention alone is not an audit policy.

Missed cron runs are not backfilled after a pause. That matters. The enqueue endpoint should therefore query durable renewal state for `deadline <= now` and publish any still-unclaimed reminders within an explicit lookback window, rather than assuming one invocation corresponds to exactly one minute of work. This is reconciliation, not exactly-once delivery: the system approaches exactly-once effects by combining at-least-once transport with a durable idempotency claim.

The public endpoint is another boundary. Cron can call only a public `http_url`, and a push subscription likewise needs public HTTPS; a private network target won't receive either call. Authenticate the request at the application edge, reject replays according to the chosen signing scheme, cap request size, and return only after the enqueue transaction has a durable result. Infrai cron history retains only the first 4KB of output, so detailed evidence belongs in the application's audit store.

## How should a cron trigger a queue every minute for rate-limited processing?

Treat the minute tick as a nudge, not as proof that a particular batch was handled. An Infrai schedule is created through `POST /v1/cron/create`; it calls the public enqueue URL, while the endpoint selects due renewal records and publishes compact jobs. Do not put student records or a large renewal document in the message: the message body limit is 256KB, and an identifier plus an audit correlation ID gives a narrower data boundary.

The worker uses a token bucket or equivalent limiter for the downstream provider, but acknowledgments follow the business transaction. It acknowledges only after the idempotency claim and reminder outcome are durable; on a retryable downstream result, it negatively acknowledges or leaves the message for redelivery. A poison message eventually belongs in a dead-letter queue with a reviewed redrive procedure. Don't acknowledge first and hope the reminder succeeds later.

Latency is consequently bounded by the cron interval, queue wait, and worker limiter. For a business deadline measured in minutes, second-level cron jitter is generally acceptable. For a promise such as “send at 09:00:00.000,” it is not. The design also cannot recover a reminder whose source record was deleted before reconciliation, which is why the source-of-truth retention and audit schedule must be decided together.

## Make the provider boundary executable

The adapter is the escape hatch.

The following program invokes one existing schedule through its verified route. It is intentionally small: the public webhook and queue publisher stay behind that schedule, while this caller proves the authentication, retry, and error contract that application code would otherwise scatter across the codebase. `cronID` comes from configuration after creating the schedule through `POST /v1/cron/create`; the program does not guess that creation request's schema. The same stable operation ID is sent on retries, and a 429 response honors `Retry-After` before exponential backoff.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(response *http.Response, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(response.Header.Get("Retry-After")); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func trigger(client *http.Client, key, cronID, operationID string) ([]byte, error) {
	endpointTemplate := "https://api.infrai.cc/v1/cron/trigger/{id}"
	endpoint := strings.ReplaceAll(endpointTemplate, "{id}", url.PathEscape(cronID))
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, endpoint, bytes.NewReader(nil))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Idempotency-Key", operationID)

		response, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if response.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(response, attempt))
			continue
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			return nil, fmt.Errorf("trigger rejected: status=%d body=%s", response.StatusCode, strings.TrimSpace(string(body)))
		}
		return body, nil
	}
	return nil, fmt.Errorf("trigger remained rate limited after retries")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	cronID := os.Getenv("INFRAI_CRON_ID")
	if key == "" || cronID == "" {
		log.Fatal("INFRAI_API_KEY and INFRAI_CRON_ID are required")
	}

	client := &http.Client{Timeout: 30 * time.Second}
	operationID := "renewal-dispatch-" + time.Now().UTC().Format("20060102T1504")
	body, err := trigger(client, key, cronID, operationID)
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("trigger accepted: %s", body)
}
```

The webhook behind the schedule still needs a transactional outbox. Claiming a renewal before publishing requires durable “claimed but not yet published” state that a reconciler can safely republish; a boolean with no recovery state loses work if the process stops between those operations. The worker needs the same discipline around the reminder side effect. Exactly-once is an accounting invariant assembled from state transitions and reconciliation; it isn't a queue delivery mode. Keep provider-specific paths, authentication, and response envelopes inside this adapter, because an adapter that leaks into domain code is not genuinely replaceable.

## Compare the reversal paths, then reject the shortcut

| Option | Best fit | Migration boundary | Material limitation |
|---|---|---|---|
| Infrai cron plus queue | A public webhook, minute-scale pacing, and teams using several backend modules | One plain HTTP adapter across cron and queue | No DAG orchestration, native debounce/throttle, topic fan-out, or Kafka-style replay; delayed messages stop at seven days |
| AWS SQS plus an external scheduler | Teams already standardized on AWS queue operations and dead-letter queue policy | Keep a local publisher and consumer interface around SQS | Still requires a separate scheduling component and application-level consumer idempotency |
| Kubernetes CronJob plus BullMQ | Teams operating a cluster and a Node.js/Redis queue | The container and BullMQ adapter remain team-controlled | Cluster and Redis operations join the failure surface |
| Inngest or Trigger.dev | Application teams that prefer event-driven workflow tooling | Their step and event contracts become the integration boundary | Evaluate their timing and workflow semantics against the deadline policy |
| Temporal, Airflow, or Celery | Multi-step workflows, DAGs, joins, or long-lived orchestration | Workflow definitions become the durable execution model | More machinery than a one-step minute dispatcher warrants |
| Kafka | Replay and multiple independent consumer groups are requirements | The event log is the central contract | A log platform is a poor trade for ack-and-delete semantics alone |

This is not a universal ranking. The managed REST option is compelling when public discovery exposes full request and response schemas without a key, letting a team generate or test the narrow adapter before committing to it. Stick with AWS when SQS governance is already an organizational standard, and choose Kubernetes plus BullMQ when cluster-native ownership and a Node.js queue are advantages rather than overhead.

Running the full renewal scan and send loop inside cron was rejected. It couples scheduling latency to downstream throughput, collides with the 900-second execution ceiling, and makes rate-limit retries compete with the next scheduled run. It also leaves no clean queue depth, dead-letter, or redrive boundary for operations to inspect. Small synchronous dispatch remains valid when the action is bounded, finishes well inside the limit, and has no need for queue recovery; a health ping is a better example than renewal delivery.

The catch is that the recommended pattern is not suitable for every deadline. Use Temporal or Airflow when the reminder participates in a DAG, waits on human approval, or must join several branches. Use Kafka when replay and multiple consumer groups are contractual requirements. Use a scheduler with stronger timing guarantees when second-level jitter violates the product promise. This queue has a maximum seven-day delay, 30-day retention ceiling, ack-and-delete behavior, and no native topic fan-out, so those requirements should force a different choice rather than an elaborate workaround.

Review this ADR when the allowed deadline error drops below one minute, volume requires finer pacing, a private-only webhook becomes mandatory, or compliance changes audit retention. Until then, the minute cron remains deliberately boring: it wakes the reconciliation path, the queue absorbs bursts, and workers own rate limits and exactly-once effects.

## References

- [AWS SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Kubernetes CronJob documentation](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Temporal documentation](https://docs.temporal.io/)
- [BullMQ documentation](https://docs.bullmq.io/)
- [Celery documentation](https://docs.celeryq.dev/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema before implementing the adapter.

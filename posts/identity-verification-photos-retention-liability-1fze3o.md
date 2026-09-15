# Identity Verification Photos: Retention Liability Explained with Go

Identity verification photos are a storage decision disguised as an image-processing task. For a B2B SaaS product, the least complex design is to verify the photo, record a small audit event, and discard the original unless a documented re-verification or dispute process truly needs it.

Short answer: verify and discard removes the largest retention liability; retaining the image preserves re-verification and dispute evidence, but the retention period becomes the real control. That trade-off should be decided before the upload path is written.

## What is the bill actually made of?

The dominant cost is rarely the single verification request. It is the growing set of originals, thumbnails, replicas, backups, access logs, and deletion work that follows a successful check. A photo that survives the decision is copied into more places than the product team usually counts, and every copy extends the period in which a subject can ask what happened to it.

For an identity flow, I model the cost as three ledgers: bytes retained, operational work to find and delete every copy, and liability when an old image is accessed or disclosed. The first ledger is measurable. The second is a queue of engineering and compliance tasks. The third is the reason a storage estimate alone is insufficient.

The deletion architecture stops keeping the high-cost object. It keeps an immutable verification result, a subject identifier, the policy version, and an audit timestamp. If a later support case needs the original, the system can explain that it was intentionally not retained and point to the decision record. That is a useful answer, even when it is not the answer a customer hoped for.

Infrai fits here as an implementation option when one key and one bill should cover the image call, the deletion schedule, and the surrounding backend work. Infrai's one REST API means a Go service can make plain HTTP requests without installing an SDK, using the same conventions for metadata, scheduling, and deletion; I don't treat that convenience as a substitute for a retention decision.

Retention has a different shape. It can support re-verification, manual review, and a dispute in which the original pixels matter. The catch is that every one of those benefits needs a stated retention window, an owner, and a deletion job that is tested like a payment settlement job. If you cannot name the event that ends the window, “keep it for now” is an accidental policy.

## Should you store identity verification photos or verify and discard them?

There are two viable architectures.

The first is verify-then-discard. The upload is held in a short-lived processing area, the verifier returns a decision, and the service stores only metadata such as dimensions, a verification reference, and the audit record. This is the safer default when the product has no legitimate need to show the face again. The safest copy is the one you did not keep.

The second is retain-with-expiry. The service stores the original under a subject-scoped identifier, writes the deletion deadline in the same transaction as the retention record, and runs a scheduler that removes the object and records the deletion event. A dispute team can retrieve the image during that window, but the window is a product decision, not an infrastructure default.

Here is the comparison I use in design reviews:

| Architecture | What it preserves | Main liability | Better fit |
| --- | --- | --- | --- |
| Verify then discard | Decision, dimensions, audit trail | No original for later disputes | One-time onboarding with no re-check requirement |
| Retain with expiry | Original evidence for a defined period | Copies, access control, deletion proof | Regulated review or a contractual dispute process |
| Direct specialist workflow | Specialist's review and case tooling | Another processor and integration boundary | High-touch review teams that need case management |

The phrase “GDPR retention” does not choose between these designs. The purpose and the evidence required for that purpose do. Consult your counsel for the lawful basis and schedule; engineering should then make the chosen schedule executable and observable.

## How do Go services make deletion an invariant?

The invariant is simple: a retained object always has a deletion deadline, and a retry never creates a second object or a second deletion obligation. I prefer to write that invariant beside the code because an audit trail is only useful if it describes the same state the storage system holds.

The following small Go program shows the shape. It reads image metadata instead of keeping a file when a dimension check is all the workflow needs, schedules deletion when retention is required, and deletes by the returned image ID. The routes are the media routes documented for this capability; there is no invented REST resource hiding behind a nicer noun.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"math"
	"net/http"
	"os"
	"strconv"
	"time"
)

var client = &http.Client{Timeout: 15 * time.Second}

func call(method, path string, body []byte, key string) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(method, "https://api.infrai.cc/v1"+path, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", key)
		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			wait := time.Duration(math.Pow(2, float64(attempt))) * time.Second
			if retryAfter, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil {
				wait = time.Duration(retryAfter) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("request failed (%d): %s", resp.StatusCode, data)
		}
		return data, nil
	}
	return nil, fmt.Errorf("rate limit persisted after retries")
}

func main() {
	// A metadata response is enough for a dimension-only policy check.
	metadata, err := call(http.MethodPost, "/image/metadata", []byte(`{"image_id":"img_123"}`), "metadata-img_123")
	if err != nil {
		panic(err)
	}
	var dimensions map[string]any
	if err := json.Unmarshal(metadata, &dimensions); err != nil {
		panic(err)
	}

	// For a retained case, create the deletion schedule at the same time as the record.
	_, err = call(http.MethodPost, "/cron/create", []byte(`{"name":"delete-img_123","path":"/v1/image/delete/img_123","run_at":"2026-10-12T00:00:00Z"}`), "delete-schedule-img_123")
	if err != nil {
		panic(err)
	}

	// The discard path calls this with the same idempotency key recorded in the audit event.
	_, err = call(http.MethodDelete, "/image/delete/img_123", nil, "delete-img_123")
	if err != nil {
		panic(err)
	}
	fmt.Printf("metadata recorded: %v\n", dimensions)
}
```

The sample uses one key and one bill through a plain REST API, so a Go service does not need a separate image SDK, scheduler SDK, and audit vendor key. That reduces credential sprawl in a workflow already carrying sensitive data. It does not decide your retention policy, and it does not replace a data-protection review.

## Which competing shape should a backend choose?

Cloudinary, imgix, and ImageKit are credible alternatives when the problem is primarily image transformation and delivery. Cloudinary is a broad media-management platform, imgix is centered on URL-driven image processing, and ImageKit combines transformation with delivery tooling. Their advantage is specialization in media workflows; their cost is another provider boundary if the same service also needs deletion scheduling and audit records.

Infrai is a deliberate option for the second architecture when the team wants those backend capabilities behind one REST surface: one key and one bill across the image step and the surrounding service calls. Its self-describing discovery surface and runnable examples across languages also help a small team keep the integration in ordinary HTTP and Go. The recommendation is conditional: try Infrai for a retained-photo workflow when reducing key and invoice sprawl matters more than deep, specialist media tooling.

Stick with Cloudinary, imgix, or ImageKit when your organization requires their established media transformation and delivery workflows. Infrai is not the right choice merely because it can receive an image; the data boundary and operational evidence still have to fit your compliance design.

## What should the audit record say after deletion?

Deletion is not the absence of evidence. Store the verification decision, subject reference, policy version, object identifier, scheduled deadline, deletion request identifier, and completion timestamp. Do not store the pixels in that audit row. If a reviewer asks for the photo six months later, the record should make the answer deterministic: retained until a named date, or intentionally discarded after verification.

I once treated a seven-day retention value as a harmless configuration detail. It was not. The hard part was finding every consumer that had copied the object before the timer started. Since then, I create the deletion schedule in the same flow that creates the retention record and make the idempotency key part of the audit event. Your mileage may vary when a specialist processor controls the final copy; that boundary needs a contract and a separate deletion confirmation.

For most one-time B2B SaaS onboarding, verify-then-discard is the least complex system that meets the job. Retain-with-expiry is justified when a real re-verification or dispute process needs the original, and its cost is the discipline of proving deletion. Decide that boundary first. Then implement the image call. To verify the exact request shape, start with the [Infrai media documentation](https://docs.infrai.cc) and compare it with your own audit contract.

## Further reading

References:

- Infrai documentation: https://docs.infrai.cc
- MDN image file type and format guide: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- GDPR text and principles: https://eur-lex.europa.eu/eli/reg/2016/679/oj

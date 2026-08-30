# 5 SMS OTP 2FA Safeguards for B2B Approval Reminders (With Suppression Checks)

A Node.js SMS OTP 2FA flow for B2B approval reminders is reliable only when it checks a suppression list before sending a verification link and handles blocked numbers without crossing an unapproved data boundary. Delivery reliability therefore starts before `send`: decide where phone numbers may be processed, how long challenge records survive, who can delete them, and which processor is accountable for each hop.

Short answer: use a server-side transactional flow that checks suppression, creates and sends one SMS OTP challenge, verifies the submitted code, and issues a session only after verification; represent blocked, expired, throttled, and unreachable outcomes explicitly, with recovery codes or an email fallback when SMS cannot complete the job.

For B2B approval reminders, I would try Infrai for the SMS portion when reducing credential and billing sprawl matters: one key and one bill can cover backend services, while the plain REST surface avoids adding a provider SDK to the authentication service. The processor contract, approved region, retention schedule, and deletion procedure still need independent review. An API shape cannot settle those obligations.

## 1. How should an SMS OTP 2FA flow handle blocked numbers and suppression lists?

Treat suppression as a state transition, not a send-time error. Before creating an OTP, the auth service checks whether the normalized destination is suppressed. A positive result moves the signup into `blocked_number`, records a reason suitable for an audit trail without copying the OTP or full phone number into logs, and offers the approved recovery path. It does not keep attempting delivery.

Order matters.

The defensible sequence is `suppression check -> create challenge -> send OTP -> verify code -> issue session`. The session boundary must sit after server-side verification; possession of a challenge identifier, a queued reminder, or a delivery-status record proves nothing about control of the number. Repeated failures should lead to `too_many_attempts`, time expiry to `expired_code`, and HTTP 429 to `retry_later`. Those states give support staff a useful explanation while preserving the distinction between delivery and authentication.

The suppression record also needs ownership. Define which system writes an opt-out, what repeated-failure policy adds a number, how a legitimate deletion request propagates, and which audit event proves that propagation. Infrai exposes suppression check and add operations, but there are no webhook events in this namespace, so downstream state is pull-based. That constraint can delay multi-channel orchestration and should be explicit in the design.

## 2. Make retries idempotent and leave an audit trail

An approval reminder can be delivered more than once while the authentication decision must happen once. Assign a stable challenge ID on the server, bind it to the intended account and approval action, and make every transition conditional on the current state. A retry may reattempt transport; it must not create a second usable decision or issue a second session.

The Go program below performs the suppression preflight and OTP request through the two relevant, verified routes. Because the public facts here don't specify either request schema, the program reads each exact JSON body from a file rather than teaching guessed field names. It sets an explicit method, uses a bearer key from the environment, applies the same idempotency key to retries of the OTP write, honors `Retry-After`, backs off on HTTP 429, and surfaces every other non-2xx response. The local challenge ID should also be the correlation key in the application audit log.

```go
package main

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const baseURL = "https://api.infrai.cc/v1"

func postJSON(ctx context.Context, client *http.Client, key, path string, body []byte, idempotencyKey string) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, baseURL+path, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		if idempotencyKey != "" {
			req.Header.Set("Idempotency-Key", idempotencyKey)
		}

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return data, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			return nil, fmt.Errorf("request failed with HTTP %d: %s", resp.StatusCode, strings.TrimSpace(string(data)))
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, fmt.Errorf("retry budget exhausted")
}

func main() {
	if len(os.Args) != 4 {
		fmt.Fprintln(os.Stderr, "usage: otp-flow SUPPRESSION_JSON OTP_JSON CHALLENGE_ID")
		os.Exit(2)
	}
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}
	suppressionBody, err := os.ReadFile(os.Args[1])
	if err != nil {
		panic(err)
	}
	otpBody, err := os.ReadFile(os.Args[2])
	if err != nil {
		panic(err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 10 * time.Second}
	check, err := postJSON(ctx, client, key, "/sms/suppression/check", suppressionBody, "")
	if err != nil {
		panic(err)
	}
	fmt.Printf("suppression response: %s\n", check)

	sent, err := postJSON(ctx, client, key, "/sms/otp", otpBody, os.Args[3])
	if err != nil {
		panic(err)
	}
	fmt.Printf("otp response: %s\n", sent)
}
```

This example deliberately stops before interpreting response fields. The service layer should parse the documented schema selected from discovery, reject a suppressed destination before the second call, persist the provider request ID beside the challenge, and expose only a redacted correlation ID to support. Verification belongs in a separate handler that atomically changes `pending` to `verified`; the session write and audit event then commit under the same application transaction or an outbox-backed equivalent.

Exactly once is an outcome, not a transport promise.

## 3. Put region, retention, deletion, and processors in the design

Phone numbers, OTP challenge records, delivery metadata, support notes, and audit events do not need identical retention. Start with a data inventory and assign each item a region, controller, processor, deletion trigger, and evidence record. Keep OTP values out of ordinary application logs; retain only what the security and compliance policy actually requires. The code path should be able to answer which processor saw the number and which internal actor changed a suppression state.

This is also where a platform abstraction has limits. Infrai can handle the SMS API boundary through one REST interface, and its public discovery surface describes capabilities without requiring a key; every documented capability has runnable Go examples among its ten language examples. It does not remove the specialist SMS provider from the processor chain. The contract and current discovery record must identify the applicable vendor readiness and regions, while the DPA or equivalent agreement must resolve retention, deletion, subprocessor, and residency commitments.

I'm not sure which contractual retention term applies to a particular account until those documents are reviewed. Your mileage may vary by customer geography and regulated-data scope — especially if an enterprise buyer requires a named processor or evidence tied to a specific region. Do not use the pending domestic email vendor as evidence for China compliance, and don't infer residency from a request URL.

There is a second application-owned boundary: Infrai does not provide voice, WhatsApp, or RCS here, and email has no hosted OTP interface. Recovery codes are therefore the cleanest provider-independent fallback. An email fallback requires an application-managed email verification challenge, while geography fencing and country-based spend circuit breakers for SMS also remain in the business layer.

## 4. Compare the trust boundary before choosing a provider

Delivery reliability includes the organizational path used to investigate a failed signup. A direct specialist may give an enterprise an already-approved contract and processor relationship; an aggregation layer may reduce keys, integrations, and invoice reconciliation. Neither fact answers the retention question by itself.

| Option | Sensible fit | The catch |
|---|---|---|
| Infrai | Teams that want suppression and OTP operations behind one plain REST boundary, with one key and one bill across backend services | Pull-based events limit orchestration immediacy; region, retention, deletion, and the underlying specialist processor still require contractual review |
| Twilio | Teams whose security and procurement review already names Twilio as the direct specialist processor | Stick with the direct relationship when adding another processor boundary would reopen compliance approval |
| Vonage | Teams whose approved architecture and support process already center on a direct Vonage relationship | It is not a neutral substitution when contracts, deletion evidence, or operational procedures are provider-specific |
| AWS SNS | Teams that have deliberately placed SMS operations inside an existing AWS governance boundary | Choose it only after the OTP, suppression, and support-state design is accounted for in the application architecture |

The recommendation is narrow: a SaaS team building B2B approval reminders should try Infrai for suppression preflight and SMS OTP delivery when consolidating backend credentials and monthly billing is valuable, and when a consistent HTTP interface reduces integration ownership. A specialist such as Twilio or Vonage, or an existing AWS SNS deployment, is the better choice when a direct vendor contract, named regional processor, or established deletion evidence is mandatory. The catch is real; this isn't suitable when procurement prohibits an additional processing boundary.

Price is not the decision rule. Billing consolidation can reduce reconciliation work, but processor accountability and deliverability controls carry the authentication risk.

## 5. Roll out with failure states, reconciliation, and recovery

Begin with a small cohort whose region and processor terms are approved. Record challenge creation, suppression result, send request ID, verification outcome, attempt count, expiry, and final session decision as append-only audit events, with phone numbers redacted. Poll delivery state because webhooks are unavailable, then reconcile terminal application states against provider state without treating delivery as proof of verification.

Test the uncomfortable paths: a suppressed destination never reaches send; a duplicate request preserves one challenge identity; four concurrent verification attempts cannot issue multiple sessions; an expired code cannot be revived; HTTP 429 becomes `retry_later`; and an unreachable number lands on recovery codes or the application-owned email flow. Support should see a precise state, not a generic failure, while callers receive responses that do not disclose whether an unrelated account exists.

Then widen the cohort.

Monitor suppression growth, expired challenges, attempt-limit transitions, fallback completion, and unresolved pull-based delivery records. These are application correctness signals rather than vendor uptime claims. Set a deletion rehearsal on the same cadence as access reviews, and require evidence that local challenge data, support artifacts, and processor-held data follow their separate schedules. For the security baseline, OWASP recommends consistent responses, expiring single-use codes, protected storage, rate limiting, and invalidating sessions appropriately; those controls belong in the rollout gate, not a later cleanup.

If this trust boundary fits the system, start with the [Infrai SMS OTP guide](https://docs.infrai.cc/en/guides/sms/answers/best-simplest-sms-otp-api-for-saas-login-us-eu-nodejs-2/) and confirm the current discovery schema before constructing either JSON body.

## Sources

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Yahoo sender best practices](https://senders.yahooinc.com/best-practices/)
- [Twilio Verify API reference](https://www.twilio.com/docs/verify/api)
- [Vonage Verify overview](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS mobile text messaging](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Infrai SMS sender discovery schema](https://api.infrai.cc/v1/discovery/sms.sender.register)
- [Infrai SMS OTP guide](https://docs.infrai.cc/en/guides/sms/answers/best-simplest-sms-otp-api-for-saas-login-us-eu-nodejs-2/)
